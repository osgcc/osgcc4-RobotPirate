module tunnel

import gl
import sdl: event, key
import math: sin, cos, abs

import text

local ToRad = math.pi / 180
local ToDeg = 180 / math.pi

import config

/*
As level progresses:
	- Randomly spawn mines
		- In patterns?
	- Randomly spawn enemies
	- Mine-dodging and dogfighting sections (where you do one to the exclusion of the other)
	- Spawn boss every some amount of distance.. and let the player kill them before resuming normal play
		- Increments difficulty level

Enemies drop items randomly
	- Health
	- Extra lives (rare)
	- Weapon upgrades
	- Shield?

Enemy types:
	- Derp. Moves slightly slower than the level itself, shoots ahead every once in a while, eventually moves off near horizon. Little health.
	- Quick. Moves forward/back and around a little way ahead of the player. Shoots at player more often than derp, but not too quick. Eventually moves off near horizon. Medium health.
	- Heavy. Not very mobile, but hovers ahead of player. Fires a lot (maybe spread shot). Must be killed. High health.
	- Boss. Not too mobile, but must be hit in weak spots. Fires a lot. Must be killed. Very high health.

Weapon types:
	- Single shooter. Slow firing rate; small bullet fired straight ahead; doesn't do much damage.
	- Double shooter. Slightly higher rate; two streams of the above.
	- Three-way shooter. Highest firing rate; shoots in a small spread, and each bullet does more damage.
	- Railgun. Boss killer. Very slow firing rate, shoots straight ahead, does a lot of damage.

SCORING:
	- Mine: 10 pts
	- Derp: 30 pts
	- Quick: 60 pts
	- Heavy: 100 pts
	- Boss: 1000 pts
*/

function main()
{
	setup()

	scope(exit)
		sdl.quit()

	loop()
}

function FreeList(T: class)
{
	T._freelist_ = null
	T.next = null
	
	if(hasMethod(T, "initialize"))
	{
		T.alloc = function alloc(this: class, vararg)
		{
			local n

			if(:_freelist_)
			{
				n = :_freelist_
				:_freelist_ = n.next
			}
			else
				n = this()

			n.initialize(vararg)
			return n
		}
	}
	else
	{
		T.alloc = function alloc(this: class)
		{
			local n

			if(:_freelist_)
			{
				n = :_freelist_
				:_freelist_ = n.next
			}
			else
				n = this()

			return n
		}
	}

	T.free = function free(this: instance)
	{
		:next = :_freelist_
		:super._freelist_ = this
	}

	return T
}

@FreeList
class Obstacle
{
	ang
	z
	colliding
	prev

	function initialize(ang: float, z: float)
	{
		:ang = ang
		:z = z
		:colliding = false
	}
}

@FreeList
class Bullet
{
	ang
	z
	dir
	prev

	function initialize(ang: float, z: float, dir: float)
	{
		:ang = ang
		:z = z
		:dir = dir * ToRad
	}
}

// ===================================================================================
// State
// ===================================================================================

local projMat2D = Vector(gl.GLfloat, 16)
local projMat3D = Vector(gl.GLfloat, 16)
local fps = 0

local quitting = false
local keys = array.new(512, false)
local keysHit = array.new(512, false)

local shipModel
local shipAng = 0.0
local dShipAng = 0
local shipMoveSpeed = 12
local shipMoveMult = 0.15
local playerHealth = 100

local shipAngLag = array.new(9, 0)
local lagNewPtr = #shipAngLag - 1
local lagOldPtr = 0
local camAng = 0.0

local levelModel
local levelSegments = 30
local levelSegmentLen = 8
local levelSpeed = 1.1
local levelOffs = 0.0
local levelZ = 0.0

local obstacleModel
local obstacles = Obstacle() // dummy sentry node
local obstacleAng = 0.0
local obsDamageAmount = 5

local playerBullets = Bullet() // dummy sentry node
local playerBulletSpeed = 2
local playerBulletDelayer = 0
local playerBulletDelay = 14
local bulletModel

// ===================================================================================
// Code
// ===================================================================================

// Game setup
function setup()
{
	setupGraphics()
	setupResources()
	setupEvents()
}

// Main game loop.
function loop()
{
	local t = time.microTime()
	local frames = 0

	while(!quitting)
	{
		doInput()
		updatePlayer()
		updateCamera()
		updateLevel()
		drawGraphics()
		
		frames++
		local dt = time.microTime() - t
		
		if(dt >= 1_000_000)
		{
			t = time.microTime()
			fps = frames / (dt / 1_000_000.0)
			frames = 0
		}
	}
}

// Set up the window and graphics mode
function setupGraphics()
{
	// Window
	sdl.init(sdl.initEverything)

	scope(failure)
		sdl.quit()

	sdl.gl.setAttribute(sdl.gl.bufferSize, 32)
	sdl.gl.setAttribute(sdl.gl.depthSize, 16)
	sdl.gl.setAttribute(sdl.gl.doubleBuffer, 1)

	if(!sdl.setVideoMode(config.winWidth, config.winHeight, 32, sdl.opengl | sdl.hwSurface))
		if(!sdl.setVideoMode(config.winWidth, config.winHeight, 32, sdl.opengl | sdl.swSurface))
			throw "Could not set video mode"

	sdl.setCaption("Tunnel")
	sdl.showCursor(false)

	// OpenGL
	gl.load()
	gl.glViewport(0, 0, config.winWidth, config.winHeight)
	gl.glShadeModel(gl.GL_FLAT)
	gl.glClearColor(0, 0, 0, 1)
	gl.glClearDepth(1)
// 	gl.glEnable(gl.GL_CULL_FACE)
	gl.glEnable(gl.GL_DEPTH_TEST)
	gl.glEnable(gl.GL_BLEND)
	gl.glEnable(gl.GL_NORMALIZE)
	gl.glBlendFunc(gl.GL_SRC_ALPHA, gl.GL_ONE_MINUS_SRC_ALPHA)
	gl.glEnable(gl.GL_LIGHTING)

	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadIdentity()
	gl.gluPerspective(45, toFloat(config.winWidth) / config.winHeight, 3, 1000)
	gl.glGetFloatv(gl.GL_PROJECTION_MATRIX, projMat3D)

	gl.glLoadIdentity()
	gl.gluOrtho2D(0, config.winWidth, 0, config.winHeight);
	gl.glScalef(1, -1, 1);
	gl.glTranslatef(0.5, -config.winHeight + 0.5, 0);
	gl.glGetFloatv(gl.GL_PROJECTION_MATRIX, projMat2D)

	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_LINE);
	gl.glLightModelfv(gl.GL_LIGHT_MODEL_AMBIENT, float4(0, 0, 0, 1))

	gl.glEnable(gl.GL_LIGHT0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_SPOT_DIRECTION, float4(0, -1, 0, 0))
// 	gl.glLightf(gl.GL_LIGHT0, gl.GL_LINEAR_ATTENUATION, 0.001);
	gl.glLightf(gl.GL_LIGHT0, gl.GL_QUADRATIC_ATTENUATION, 0.1);
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -6, -26, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_DIFFUSE, float4(0, 10, 0, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_AMBIENT, float4(0, 10, 0, 1))

	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
	
	text.setup()
}

// Load resources
function setupResources()
{
	levelModel = makeTunnelPiece(20)
	shipModel = loadModel("models/ship.obj")
	obstacleModel = loadModel("models/obstacle.obj")
	bulletModel = loadModel("models/bullet.obj")
}

// Set up event handlers
function setupEvents()
{
	event.setHandler$ event.quit, \{ quitting = true }
	event.setHandler$ event.key, \pressed, sym, mod { keys[sym] = pressed; keysHit[sym] = pressed }
}

// Input/player control
function doInput()
{
	keysHit.fill(false)
	event.poll()

	if(keys[key.escape])
		quitting = true

	if(keys[key.left] && !keys[key.right])
		dShipAng -= 1
	else if(keys[key.right] && !keys[key.left])
		dShipAng += 1
	else
	{
		if(dShipAng > 0)
			dShipAng -= 1
		else if(dShipAng < 0)
			dShipAng += 1
	}

	if(playerBulletDelayer == 0 && keys[key.space])
	{
		playerBulletDelayer = playerBulletDelay
		insertIntoList(Bullet.alloc(shipAng, -30.0, 0.0), playerBullets)
// 		insertIntoList(Bullet.alloc(shipAng, -30.0, 75.0), playerBullets)
// 		insertIntoList(Bullet.alloc(shipAng, -30.0, -75.0), playerBullets)
	}
}

// Player
function updatePlayer()
{
	dShipAng = clamp(dShipAng, -shipMoveSpeed, shipMoveSpeed)
	shipAng = wrapAngle(shipAng + dShipAng * shipMoveMult)

	if(playerBulletDelayer > 0)
		playerBulletDelayer--

	for(local bul = playerBullets; bul && bul.next !is null; bul = bul.next)
	{
		local b = bul.next
		
		local dAng, dZ = sin(b.dir) * playerBulletSpeed, cos(b.dir) * playerBulletSpeed

		b.ang = wrapAngle(b.ang + dAng)
		b.z -= dZ

		if(b.z < -180)
		{
			removeFromList(b)
			b.free()
			continue
		}

		// Check for collision with obstacles
		for(local obs = obstacles.next; obs !is null; obs = obs.next)
		{
			if(abs(b.z - obs.z) < 2 && angDiff(b.ang, obs.ang) < 7)
			{
				removeFromList(obs)
				obs.free()
				removeFromList(b)
				b.free()
				break
			}
		}
	}
}

// Camera
function updateCamera()
{
	camAng = wrapAngle(camAng + shipAngLag[lagOldPtr] * shipMoveMult)
	shipAngLag[lagNewPtr] = dShipAng
	lagNewPtr++
	if(lagNewPtr == #shipAngLag)
		lagNewPtr = 0
	lagOldPtr++
	if(lagOldPtr == #shipAngLag)
		lagOldPtr = 0
}

// Level stuff
function updateLevel()
{
	levelZ += levelSpeed
	
	if(levelZ % 20 < 1)
		insertIntoList(Obstacle.alloc(math.frand(360.0), -240.0), obstacles)

	levelOffs += levelSpeed

	if(levelOffs > levelSegmentLen)
		levelOffs -= levelSegmentLen

	obstacleAng = wrapAngle(obstacleAng + 5)

	for(local obs = obstacles; obs && obs.next !is null; obs = obs.next)
	{
		local o = obs.next

		o.z += levelSpeed
		
		if(abs(o.z + 30) < 2 && angDiff(o.ang, shipAng) < 15)
		{
			// TODO: show effect
			damagePlayer(obsDamageAmount, true)
			removeFromList(o)
			o.free()
			continue
		}

		if(o.z > 0)
		{
			removeFromList(o)
			o.free()
			continue
		}
	}
}

// Graphics
function drawGraphics()
{
	gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
		draw3D()
		draw2D()
	sdl.gl.swapBuffers()
}

function draw3D()
{
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadMatrixf(projMat3D)
	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glEnable(gl.GL_LIGHTING)

	gl.glLoadIdentity()
	gl.glRotatef(-camAng, 0, 0, 1)

	drawLevel()
	drawObstacles()
	drawPlayerBullets()
	drawPlayer()
}

function draw2D()
{
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadMatrixf(projMat2D)
	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glDisable(gl.GL_LIGHTING)
	gl.glLoadIdentity()
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
	text.drawText(0, 20, "FPS:{} MEM:{}", toInt(fps), gc.allocated())
	text.drawText(0, 45, "ANG:{}", shipAng)

	drawHealthBar()
}

function drawHealthBar()
{
	gl.glPushMatrix()
		gl.glTranslatef(config.winWidth / 2, config.winHeight - 30, 0)
		gl.glBegin(gl.GL_LINE_STRIP)
			gl.glVertex2f(-80, 10)
			gl.glVertex2f(80, 10)
			gl.glVertex2f(80, -10)
			gl.glVertex2f(-80, -10)
			gl.glVertex2f(-80, 10)
		gl.glEnd()

		local temp = playerHealth / 100.0
		gl.glColor3f(1 - temp, temp, 0)
		gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_FILL);

		gl.glScalef(temp, 1, 1)
		gl.glBegin(gl.GL_QUADS)
			gl.glVertex2f(-80, 10)
			gl.glVertex2f(80, 10)
			gl.glVertex2f(80, -10)
			gl.glVertex2f(-80, -10)
		gl.glEnd()

		gl.glColor3f(1, 1, 1)
		gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_LINE);

	gl.glPopMatrix()
}

function drawLevel()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.05, 0.05, 0.05, 1)

	gl.glPushMatrix()
		gl.glTranslatef(0, 0, levelOffs)
		for(i: 0 .. levelSegments)
		{
			gl.glTranslatef(0, 0, -levelSegmentLen)
			gl.glCallList(levelModel)
		}
	gl.glPopMatrix()
}

function drawObstacles()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 0, 0, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.5, 0, 0, 1)

	for(local obs = obstacles.next; obs !is null; obs = obs.next)
	{
		gl.glPushMatrix()
			gl.glRotatef(obs.ang, 0, 0, 1)
			gl.glTranslatef(0, -9, obs.z)
			gl.glRotatef(obstacleAng, 0, 1, 0)
			gl.glCallList(obstacleModel)
		gl.glPopMatrix()
	}
}

function drawPlayerBullets()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(0, 0, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0, 0, 1, 1)

	for(local bullet = playerBullets.next; bullet !is null; bullet = bullet.next)
	{
		gl.glPushMatrix()
			gl.glRotatef(bullet.ang, 0, 0, 1)
			gl.glTranslatef(0, -9, bullet.z)
			gl.glCallList(bulletModel)
		gl.glPopMatrix()
	}
}

function drawPlayer()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.8, 0.8, 0.8, 1)

	gl.glPushMatrix()
		gl.glRotatef(shipAng, 0, 0, 1)
		gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -8, -23, 1))
		gl.glTranslatef(0, -9, -30)
		gl.glCallList(shipModel)
	gl.glPopMatrix()
}

function damagePlayer(amt: int, explosive: bool = false)
{
	playerHealth = clamp(playerHealth - amt, 0, 100)
	local temp = playerHealth / 10.0
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_DIFFUSE, float4(10 - temp, temp, 0, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_AMBIENT, float4(10 - temp, temp, 0, 1))
}

// ===================================================================================
// Utilities
// ===================================================================================

// Turns 4 floats into a float vector suitable to be passed to many GL funcs.
{
	local tmp = Vector(gl.GLfloat, 4)
	function float4(x, y, z, w)
	{
		tmp[0] = x
		tmp[1] = y
		tmp[2] = z
		tmp[3] = w
		return tmp
	}
}

// Makes a piece of the tunnel. Returns it as a GL display list.
function makeTunnelPiece(numSegments: int)
{
	local num = gl.glGenLists(1)

	gl.glNewList(num, gl.GL_COMPILE)
		gl.glBegin(gl.GL_QUADS)
			local p1x, p1y = 0, 10
			local div = 2 * math.pi / numSegments
			local divhalf = div / 2.0

			for(i: 0 .. numSegments)
			{
				local ang = div * (i + 1)
				local s, c = sin(ang), cos(ang)
				local p2x, p2y = s * 10, c * 10
				ang -= divhalf
				local nx, ny = -s, -c

				gl.glNormal3f(nx, ny, 0)
				gl.glVertex3f(p1x, p1y, levelSegmentLen)
				gl.glVertex3f(p1x, p1y, 0)
				gl.glVertex3f(p2x, p2y, 0)
				gl.glVertex3f(p2x, p2y, levelSegmentLen)

				p1x, p1y = p2x, p2y
			}
		gl.glEnd()
	gl.glEndList()

	return num
}

// Loads a wavefront OBJ file and returns it as a GL display list.
function loadModel(filename: string)
{
	local verts = []
	local faces = []

	foreach(line; io.lines(filename))
	{
		if(line.startsWith("v "))
			verts.append$ line[2..].split(" ").apply(toFloat)
		else if(line.startsWith("f "))
			faces.append$ line[2..].split(" ").apply(\s -> toInt(s[..s.find('/')]))
	}

	local num = gl.glGenLists(1)
	
	gl.glNewList(num, gl.GL_COMPILE)
		local prim = 3

		gl.glBegin(gl.GL_TRIANGLES)
			foreach(face; faces)
			{
				if(#face == 3 && prim == 4)
				{
					gl.glEnd()
					gl.glBegin(gl.GL_TRIANGLES)
				}
				else if(#face == 4 && prim == 3)
				{
					gl.glEnd()
					gl.glBegin(gl.GL_QUADS)
				}

				gl.glNormal3f$ calcNormal$ verts[face[0] - 1], verts[face[1] - 1], verts[face[2] - 1]

				foreach(vert; face)
					gl.glVertex3f(verts[vert - 1].expand())
			}
		gl.glEnd()
	gl.glEndList()

	return num
}

// Saturates v to the range [min, max].
function clamp(v, min, max) =
	v < min ? min : v > max ? max : v

// Wraps v to the range [min, max).
function wrap(v, min, max)
{
	if(v < min)
	{
		local d = max - min
		while(v < min)
			v += d
	}
	else if(v >= max)
	{
		local d = max - min
		while(v >= max)
			v -= d
	}

	return v
}

// Keeps v in the range [0, 360).
function wrapAngle(v)
{
	if(v < 0)
	{
		while(v < 0)
			v += 360
	}
	else if(v >= 360)
	{
		while(v >= 360)
			v -= 360
	}
	
	return v
}

// Calculates the normal of the plane described by the points p, q, and r.
function calcNormal(p, q, r)
{
	local a1, a2, a3 = q[0] - p[0], q[1] - p[1], q[2] - p[2]
	local b1, b2, b3 = r[0] - p[0], r[1] - p[1], r[2] - p[2]
	return cross(a1, a2, a3, b1, b2, b3)
}

// Computes the cross product of 2 vectors.
function cross(a1, a2, a3, b1, b2, b3)
	return a2 * b3 - a3 * b2, a3 * b1 - a1 * b3, a1* b2 - a2 * b1

// Computes the difference between two angles (the smallest distance in degrees).
function angDiff(a1, a2) =
	abs((a1 + 180 - a2) % 360  - 180)

// Inserts the object o at the head of the doubly linked list l.
function insertIntoList(o, l)
{
	o.next = l.next
	if(o.next)
		o.next.prev = o
	o.prev = l
	l.next = o
}

// Removes the object o from its list.
function removeFromList(o)
{
	o.prev.next = o.next
	if(o.next)
		o.next.prev = o.prev
}