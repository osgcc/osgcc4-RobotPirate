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

	function genOneList()
	{
		gl.glGenLists(1, tmp)
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

// 	gl.glClearColor(26/255.0, 26/255.0, 26/255.0, 1)
	gl.glLightModelfv(gl.GL_LIGHT_MODEL_AMBIENT, float4(0, 0, 0, 1))
	gl.glEnable(gl.GL_LIGHT0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -7, -30, 1))
// 	gl.glLightfv(gl.GL_LIGHT0, gl.GL_AMBIENT,  float4(0.3, 0.3, 0.3, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_SPOT_DIRECTION, float4(0, -1, 0, 0))
	gl.glLightf(gl.GL_LIGHT0, gl.GL_QUADRATIC_ATTENUATION, 0.02);
// 	gl.glLightf(gl.GL_LIGHT0, gl.GL_SPOT_CUTOFF, 45)
// 	gl.glLightf(gl.GL_LIGHT0, gl.GL_SPOT_EXPONENT, 24)

	local rot = 0.0
	local d = makeDerp(30)
	local n = 40
	local offs = 0

	while(!quitting)
	{
		keysHit.fill(false)
		event.poll()

		if(keys[key.escape])
			quitting = true

		if(keys[key.space])
		{
			offs += 0.5
			if(offs > 8)
				offs -= 8
		}

		if(keys[key.up])
			rot -= 1
		else if(keys[key.down])
			rot += 1

		gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
			gl.glLoadIdentity()
			gl.glRotatef(rot, 1, 0, 0)
			gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -6, -30, 1))

			gl.glPushMatrix()
				gl.glTranslatef(0, 0, offs)
				for(i: 0 .. n)
				{
					gl.glTranslatef(0, 0, -3)
					gl.glCallList(d)
				}
			gl.glPopMatrix()
		sdl.gl.swapBuffers()
	}
}

function makeDerp(numSegments: int)
{
	local num = genOneList()

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