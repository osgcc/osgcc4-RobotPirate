module tunnel

import gl
import sdl: event, key
import math: sin, cos, abs

import config

/*
- player/camera/movement
- obstacles
- bullets
- enemies
- DERP
*/

function main()
{
	setup()

	scope(exit)
		sdl.quit()

	loop()
}

class Obstacle
{
	freelist // static

	ang
	z
	colliding
	next
	
	function get(this: class, ang: float, z: float)
	{
		local ret

		if(:freelist)
		{
			ret = :freelist
			:freelist = ret.next
		}
		else
			ret = Obstacle()

		ret.ang = ang
		ret.z = z
		ret.colliding = false
		return ret
	}
	
	function free(this: class, obs: Obstacle)
	{
		obs.next = :freelist
		:freelist = obs
	}
}

// ===================================================================================
// State
// ===================================================================================

local quitting = false
local keys = array.new(512, false)
local keysHit = array.new(512, false)

local shipModel
local shipAng = 0.0
local dShipAng = 0
local shipMoveSpeed = 12
local shipMoveMult = 0.15

local shipAngLag = array.new(9, 0)
local lagNewPtr = #shipAngLag - 1
local lagOldPtr = 0
local camAng = 0.0

local levelModel
local levelSegments = 60
local levelSegmentLen = 4
local levelSpeed = 1.1
local levelOffs = 0.0
local levelZ = 0.0

local obstacleModel
local obstacles = Obstacle() // dummy sentry node
local obstacleAng = 0.0

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
	while(!quitting)
	{
		doInput()
		updatePlayer()
		updateCamera()
		updateLevel()
		drawGraphics()
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
	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_LINE);
	gl.glLightModelfv(gl.GL_LIGHT_MODEL_AMBIENT, float4(0.1, 0.1, 0.1, 1))

	gl.glEnable(gl.GL_LIGHT0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_SPOT_DIRECTION, float4(0, -1, 0, 0))
	gl.glLightf(gl.GL_LIGHT0, gl.GL_QUADRATIC_ATTENUATION, 0.02);
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -6, -26, 1))

	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
}

// Load resources
function setupResources()
{
	levelModel = makeTunnelPiece(30)
	shipModel = loadModel("models/ship.obj")
	obstacleModel = loadModel("models/obstacle.obj")

	local obs = obstacles

	for(i: 0 .. 180, 30)
	{
		for(j: 0 .. 360, 360/15)
		{
			obs.next = Obstacle.get(toFloat(j), -toFloat(i))
			obs = obs.next
		}
	}
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
}

// Player
function updatePlayer()
{
	dShipAng = clamp(dShipAng, -shipMoveSpeed, shipMoveSpeed)
	shipAng = (shipAng + dShipAng * shipMoveMult) % 360
}

// Camera
function updateCamera()
{
	camAng = (camAng + shipAngLag[lagOldPtr] * shipMoveMult) % 360
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
	levelOffs += levelSpeed

	if(levelOffs > levelSegmentLen)
		levelOffs -= levelSegmentLen

	obstacleAng = (obstacleAng + 5) % 360

	for(local obs = obstacles; obs.next !is null; obs = obs.next)
	{
		local o = obs.next

		o.z += levelSpeed
		o.colliding = o.z > -32 && o.z < -28 && angDiff(o.ang, shipAng) < 15

		if(o.z > 0)
		{
			//o.ang = math.frand(360.0)
			o.z = -180
		}
	}
}

// Graphics
function drawGraphics()
{
	gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
		gl.glLoadIdentity()
		gl.glRotatef(-camAng, 0, 0, 1)

		gl.glPushMatrix()
			gl.glTranslatef(0, 0, levelOffs)
			for(i: 0 .. levelSegments)
			{
				gl.glTranslatef(0, 0, -levelSegmentLen)
				gl.glCallList(levelModel)
			}
		gl.glPopMatrix()

		for(local obs = obstacles.next; obs !is null; obs = obs.next)
		{
			if(obs.colliding)
			{
				gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 0, 0, 1)
				gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(1, 0, 0, 1)
			}
			else
			{
				gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
				gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0, 0, 0, 1)
			}

			gl.glPushMatrix()
				gl.glRotatef(obs.ang, 0, 0, 1)
				gl.glTranslatef(0, -9, obs.z)
				gl.glRotatef(obstacleAng, 0, 1, 0)
				gl.glCallList(obstacleModel)
			gl.glPopMatrix()
		}

		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0, 0, 0, 1)

		gl.glPushMatrix()
			gl.glRotatef(shipAng, 0, 0, 1)
			gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -6, -26, 1))
			gl.glTranslatef(0, -9, -30)
			gl.glCallList(shipModel)
		gl.glPopMatrix()
	sdl.gl.swapBuffers()
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
	else if(v > max)
	{
		local d = max - min
		while(v >= max)
			v -= d
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