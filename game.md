module game

import gl
import sdl: event, key
import math: sin, cos

{
	local tmp = Vector("f32", 4)
	function float4(x, y, z, w)
	{
		tmp[0] = x
		tmp[1] = y
		tmp[2] = z
		tmp[3] = w
		return tmp
	}
}

function loop()
{
	local quitting = false
	event.setHandler$ event.quit, \{ quitting = true }

	local keys = array.new(512, false)
	event.setHandler$ event.key, \pressed, sym, mod { keys[sym] = pressed }

	gl.glEnable(gl.GL_LIGHT0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(-1, -1, 0, 0))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_AMBIENT,  float4(0.3, 0.3, 0.3, 1))

	local rot = 0.0

	while(!quitting)
	{
		event.poll()

		if(keys[key.escape])
			quitting = true

		if(keys[key.left])
			rot = (rot + 5) % 360
		else if(keys[key.right])
			rot = (rot - 5) % 360

		gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
			gl.glLoadIdentity()

			gl.glPushMatrix()
				gl.glTranslatef(0, 0, -30)
				gl.glRotatef(rot, 0, 1, 0)

				derp(12)
			gl.glPopMatrix()
		sdl.gl.swapBuffers()
	}
}

function derp(numSegments: int)
{
	gl.glBegin(gl.GL_QUADS)
		local p1x, p1y = 0, 10
		local div = 2 * math.pi / numSegments
		local divhalf = div / 2.0

		for(i: 0 .. numSegments)
		{
			local ang = div * (i + 1)
			local p2x, p2y = sin(ang) * 10, cos(ang) * 10
			ang -= divhalf
			local nx, ny = sin(ang), cos(ang)

			gl.glNormal3f(nx, ny, 0)
			gl.glVertex3f(p1x, p1y, 8)
			gl.glVertex3f(p1x, p1y, 0)
			gl.glVertex3f(p2x, p2y, 0)
			gl.glVertex3f(p2x, p2y, 8)

			p1x, p1y = p2x, p2y
		}
	gl.glEnd()
}