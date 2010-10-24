module game

import gl
import sdl: event, key
import math: sin, cos

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

{
	local tmp = Vector(gl.GLuint, 1)

	function genOneBuffer()
	{
		gl.glGenBuffers(1, tmp)
		return tmp[0]
	}
}

function loop()
{
	local quitting = false
	event.setHandler$ event.quit, \{ quitting = true }

	local keys = array.new(512, false)
	local keysHit = array.new(512, false)
	event.setHandler$ event.key, \pressed, sym, mod { keys[sym] = pressed; keysHit[sym] = pressed }

// 	gl.glLightModelfv(gl.GL_LIGHT_MODEL_AMBIENT, float4(0, 0, 0, 1))
	gl.glEnable(gl.GL_LIGHT0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -7, -30, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_SPOT_DIRECTION, float4(0, -1, 0, 0))
	gl.glLightf(gl.GL_LIGHT0, gl.GL_QUADRATIC_ATTENUATION, 0.02);

// 	gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_LINE);

	local shipAng = 0.0
	local dShipAng = 0
	local shipAngLag = array.new(8, 0)
	local lagNewPtr = #shipAngLag - 1
	local lagOldPtr = 0
	local camAng = 0.0

	local d = makeDerp(30)
	local n = 40
	local offs = 0

	local ship = loadModel("models/ship.obj")

	while(!quitting)
	{
		keysHit.fill(false)
		event.poll()

		if(keys[key.escape])
			quitting = true

		offs += 0.5
		if(offs > 3)
			offs -= 3
			
		shipAngLag[lagNewPtr] = dShipAng
		lagNewPtr = (lagNewPtr + 1) % #shipAngLag

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

		dShipAng = clamp(dShipAng, -12, 12)
		shipAng = (shipAng + dShipAng * 0.125) % 360
		camAng = (camAng + shipAngLag[lagOldPtr] * 0.125) % 360
		lagOldPtr = (lagOldPtr + 1) % #shipAngLag

		gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
			gl.glLoadIdentity()
			gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -6, -26, 1))
			gl.glRotatef(camAng, 0, 0, 1)

			gl.glPushMatrix()
				gl.glTranslatef(0, 0, offs)
				for(i: 0 .. n)
				{
					gl.glTranslatef(0, 0, -3)
					gl.glCallList(d)
				}
			gl.glPopMatrix()

			gl.glPushMatrix()
				gl.glRotatef(-shipAng, 0, 0, 1)
				gl.glTranslatef(0, -9, -30)
				gl.glCallList(ship)
			gl.glPopMatrix()
		sdl.gl.swapBuffers()
	}
}

function makeDerp(numSegments: int)
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

function calcNormal(p, q, r)
{
	local a1, a2, a3 = q[0] - p[0], q[1] - p[1], q[2] - p[2]
	local b1, b2, b3 = r[0] - p[0], r[1] - p[1], r[2] - p[2]
	return a2 * b3 - a3 * b2, a3 * b1 - a1 * b3, a1* b2 - a2 * b1
}

function clamp(v, min, max) = v < min ? min : v > max ? max : v