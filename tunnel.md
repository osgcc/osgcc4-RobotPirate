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
	- Railgun. Boss killer. Very slow firing rate, shoots straight ahead, only lasts a frame, does a lot of damage.

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
class Mine
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
	damage

	function initialize(ang: float, z: float, dir: float, damage: int)
	{
		:ang = ang
		:z = z
		:dir = dir * ToRad
		:damage = damage
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

local playerMoveSpeed = 12
local playerMoveMult = 0.15
local playerWeaponChangeDelay = 20
local playerModel
local playerAng
local dPlayerAng
local playerHealth
local playerWeaponLevel
local playerWeapon
local playerScore
local playerFiringRailgun

local playerAngLag = array.new(9, 0)
local lagNewPtr
local lagOldPtr
local camAng

local levelSegments = 30
local levelSegmentLen = 8
local levelSpeed = 1.1
local levelModel
local levelOffs
local levelZ

local mineDamageAmount = 5
local mines = Mine() // dummy sentry node
local mineModel
local mineAng

local playerBulletSpeed = 2
local playerBulletDelay = 14
local playerBullets = Bullet() // dummy sentry node
local playerBulletDelayer
local bulletModel

local playerWeaponDescs =
[
	{
		level = 0
		name = "shooter1"
		fireRate = 14
		damage = 1

		function fire()
		{
			insertIntoList(Bullet.alloc(playerAng, -30.0, 0.0, :damage), playerBullets)
		}
	}

	{
		level = 1
		name = "shooter2"
		fireRate = 12
		damage = 2

		function fire()
		{
			insertIntoList(Bullet.alloc(wrapAngle(playerAng + 3), -30.0, 0.0, :damage), playerBullets)
			insertIntoList(Bullet.alloc(wrapAngle(playerAng - 3), -30.0, 0.0, :damage), playerBullets)
		}
	}

	{
		level = 2
		name = "shooter3"
		fireRate = 10
		damage = 3

		function fire()
		{
			insertIntoList(Bullet.alloc(playerAng, -30.0, 0.0, :damage), playerBullets)
			insertIntoList(Bullet.alloc(wrapAngle(playerAng + 3), -30.0, 15.0, :damage), playerBullets)
			insertIntoList(Bullet.alloc(wrapAngle(playerAng - 3), -30.0, -15.0, :damage), playerBullets)
		}
	}
	
	{
		level = 3
		name = "railgun"
		fireRate = 60
		damage = 50

		function fire()
		{
			playerFiringRailgun = 4
		}
	}
]

// ===================================================================================
// Code
// ===================================================================================

function resetState()
{
	playerAng = 0.0
	dPlayerAng = 0
	playerHealth = 100
	playerFiringRailgun = 0
	playerAngLag.fill(0)
	lagNewPtr = #playerAngLag - 1
	lagOldPtr = 0
	camAng = 0.0
	mineAng = 0.0
	levelOffs = 0.0
	levelZ = 0.0

	while(mines.next)
	{
		local m = mines.next
		removeFromList(m)
		m.free()
	}

	while(playerBullets.next)
	{
		local b = playerBullets.next
		removeFromList(b)
		b.free()
	}

	playerBulletDelayer = 0
	playerWeaponLevel = 3 // TODO: default to 0
	playerWeapon = playerWeaponDescs[0]
	playerScore = 0
}

// Game setup
function setup()
{
	setupGraphics()
	setupResources()
	setupEvents()
	resetState()
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
	playerModel = loadModel("models/ship.obj")
	mineModel = loadModel("models/obstacle.obj")
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
		dPlayerAng -= 1
	else if(keys[key.right] && !keys[key.left])
		dPlayerAng += 1
	else
	{
		if(dPlayerAng > 0)
			dPlayerAng -= 1
		else if(dPlayerAng < 0)
			dPlayerAng += 1
	}
	
	local weaponChange = playerWeapon.level

	if(keysHit[key.z])
		weaponChange = 0
	else if(keysHit[key.x] && playerWeapon.level != 1)
		weaponChange = 1
	else if(keysHit[key.c] && playerWeapon.level != 2)
		weaponChange = 2
	else if(keysHit[key.v] && playerWeapon.level != 3)
		weaponChange = 3

	if(weaponChange <= playerWeaponLevel && weaponChange != playerWeapon.level)
		changeWeapon(weaponChange)

	if(playerBulletDelayer == 0 && keys[key.space])
	{
		playerBulletDelayer = playerWeapon.fireRate
		playerWeapon.fire()
	}
}

function changeWeapon(idx: int)
{
	playerWeapon = playerWeaponDescs[idx]
	playerBulletDelayer = math.max(playerWeaponChangeDelay, playerWeapon.fireRate)
}

// Player
function updatePlayer()
{
	dPlayerAng = clamp(dPlayerAng, -playerMoveSpeed, playerMoveSpeed)
	playerAng = wrapAngle(playerAng + dPlayerAng * playerMoveMult)

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

		// Check for collision with mines
		for(local mine = mines.next; mine !is null; mine = mine.next)
		{
			if(abs(b.z - mine.z) < 2 && angDiff(b.ang, mine.ang) < 7)
			{
				removeFromList(mine)
				mine.free()
				removeFromList(b)
				b.free()
				break
			}
		}
	}

	if(playerFiringRailgun)
	{
		// Check for collision with mines
		for(local mine = mines; mine && mine.next !is null; mine = mine.next)
		{
			local m = mine.next

			if(m.z < -30 && angDiff(playerAng, m.ang) < 7)
			{
				removeFromList(m)
				m.free()
			}
		}

		playerFiringRailgun--
	}
}

// Camera
function updateCamera()
{
	camAng = wrapAngle(camAng + playerAngLag[lagOldPtr] * playerMoveMult)
	playerAngLag[lagNewPtr] = dPlayerAng
	lagNewPtr++
	if(lagNewPtr == #playerAngLag)
		lagNewPtr = 0
	lagOldPtr++
	if(lagOldPtr == #playerAngLag)
		lagOldPtr = 0
}

// Level stuff
function updateLevel()
{
	levelZ += levelSpeed
	
	if(levelZ % 20 < 1)
		insertIntoList(Mine.alloc(math.frand(360.0), -240.0), mines)

	levelOffs += levelSpeed

	if(levelOffs > levelSegmentLen)
		levelOffs -= levelSegmentLen

	mineAng = wrapAngle(mineAng + 5)

	for(local mine = mines; mine && mine.next !is null; mine = mine.next)
	{
		local m = mine.next

		m.z += levelSpeed
		
		if(abs(m.z + 30) < 2 && angDiff(m.ang, playerAng) < 15)
		{
			// TODO: show effect
			damagePlayer(mineDamageAmount, true)
			removeFromList(m)
			m.free()
			continue
		}

		if(m.z > 0)
		{
			removeFromList(m)
			m.free()
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
	drawMines()
	drawPlayer()
	drawPlayerBullets()
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

function drawMines()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 0, 0, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.5, 0, 0, 1)

	for(local mine = mines.next; mine !is null; mine = mine.next)
	{
		gl.glPushMatrix()
			gl.glRotatef(mine.ang, 0, 0, 1)
			gl.glTranslatef(0, -9, mine.z)
			gl.glRotatef(mineAng, 0, 1, 0)
			gl.glCallList(mineModel)
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

	if(playerFiringRailgun)
	{
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 0, 1)
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(1, 1, 0, 1)
		
		gl.glPushMatrix()
			gl.glRotatef(playerAng, 0, 0, 1)
			gl.glTranslatef(0, -9, -30)
			gl.glBegin(gl.GL_LINE_STRIP)
				for(i: 0 .. 240, 20)
				{
					gl.glVertex3f(1, 0, -i)
					gl.glVertex3f(-1, 0, -i - 10)
				}
			gl.glEnd()
		gl.glPopMatrix()
	}
}

function drawPlayer()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.8, 0.8, 0.8, 1)

	gl.glPushMatrix()
		gl.glRotatef(playerAng, 0, 0, 1)
		gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -8, -23, 1))
		gl.glTranslatef(0, -9, -30)
		gl.glCallList(playerModel)
	gl.glPopMatrix()
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
	text.drawText(0, 45, "ANG:{}", playerAng)

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