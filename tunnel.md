module tunnel

import gl
import sdl: event, key
import math: sin, cos

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

// ===================================================================================
// State
// ===================================================================================

local quitting = false
local keys = array.new(512, false)
local keysHit = array.new(512, false)

local shipModel
local shipAng = 0.0
local dShipAng = 0
local shipAngLag = array.new(9, 0)
local lagNewPtr = #shipAngLag - 1
local lagOldPtr = 0
local camAng = 0.0

local levelModel
local levelSegments = 40
local levelOffs = 0

// ===================================================================================
// Main functions
// ===================================================================================

// Game setup
function setup()
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
	gl.glEnable(gl.GL_CULL_FACE)
	gl.glEnable(gl.GL_DEPTH_TEST)
	gl.glEnable(gl.GL_BLEND)
	gl.glEnable(gl.GL_NORMALIZE)
	gl.glBlendFunc(gl.GL_SRC_ALPHA, gl.GL_ONE_MINUS_SRC_ALPHA)
	gl.glEnable(gl.GL_LIGHTING)
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadIdentity()
	gl.gluPerspective(45, toFloat(config.winWidth) / config.winHeight, 3, 1000)
	gl.glMatrixMode(gl.GL_MODELVIEW)
// 	gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_LINE);
// 	gl.glLightModelfv(gl.GL_LIGHT_MODEL_AMBIENT, float4(0, 0, 0, 1))
	gl.glEnable(gl.GL_LIGHT0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_SPOT_DIRECTION, float4(0, -1, 0, 0))
	gl.glLightf(gl.GL_LIGHT0, gl.GL_QUADRATIC_ATTENUATION, 0.02);

	// Resources
	levelModel = makeTunnelPiece(30)
	shipModel = loadModel("models/ship.obj")
	
	// Event handlers
	event.setHandler$ event.quit, \{ quitting = true }
	event.setHandler$ event.key, \pressed, sym, mod { keys[sym] = pressed; keysHit[sym] = pressed }
}

// Input/player control
local function doInput()
{
	keysHit.fill(false)
	event.poll()

	if(keys[key.escape])
		quitting = true

	if(keys[key.left] && !keys[key.right])
		dShipAng += 1
	else if(keys[key.right] && !keys[key.left])
		dShipAng -= 1
	else
	{
		if(dShipAng > 0)
			dShipAng -= 1
		else if(dShipAng < 0)
			dShipAng += 1
	}
}

// Player
local function updatePlayer()
{
	dShipAng = clamp(dShipAng, -12, 12)
	shipAng = (shipAng + dShipAng * 0.125) % 360
}

// Camera
local function updateCamera()
{
	camAng = (camAng + shipAngLag[lagOldPtr] * 0.125) % 360
	shipAngLag[lagNewPtr] = dShipAng
	lagNewPtr++
	if(lagNewPtr == #shipAngLag)
		lagNewPtr = 0
	lagOldPtr++
	if(lagOldPtr == #shipAngLag)
		lagOldPtr = 0
}

// Level stuff
local function updateLevel()
{
	levelOffs += 0.5
	if(levelOffs > 3)
		levelOffs -= 3
}

// Graphics
local function drawGraphics()
{
	gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
		gl.glLoadIdentity()
		gl.glRotatef(camAng, 0, 0, 1)

		gl.glPushMatrix()
			gl.glTranslatef(0, 0, levelOffs)
			for(i: 0 .. levelSegments)
			{
				gl.glTranslatef(0, 0, -3)
				gl.glCallList(levelModel)
			}
		gl.glPopMatrix()

		gl.glPushMatrix()
			gl.glRotatef(-shipAng, 0, 0, 1)
			gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -6, -26, 1))
			gl.glTranslatef(0, -9, -30)
			gl.glCallList(shipModel)
		gl.glPopMatrix()
	sdl.gl.swapBuffers()
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
				gl.glVertex3f(p1x, p1y, 3)
				gl.glVertex3f(p1x, p1y, 0)
				gl.glVertex3f(p2x, p2y, 0)
				gl.glVertex3f(p2x, p2y, 3)

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
			faces.append$ line[2..].split(" ").apply(\s -> s.split("//").apply(toInt))
	}

	local num = gl.glGenLists(1)
	
	gl.glNewList(num, gl.GL_COMPILE)
		gl.glBegin(gl.GL_QUADS)
			foreach(face; faces)
			{
				gl.glNormal3f$ calcNormal$ verts[face[0][0] - 1], verts[face[1][0] - 1], verts[face[2][0] - 1]

				foreach(vert; face)
					gl.glVertex3f(verts[vert[0] - 1].expand())
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