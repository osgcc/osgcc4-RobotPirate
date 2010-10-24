module text

import gl

local draw
local chars = array.new(128, null)
local sbuf = StringBuffer()

function drawChar(c: char)
{
	local i = toInt(c)

	if(i < #chars && chars[i] !is null)
		gl.glCallList(chars[i])
}

function setup()
{
	for(i: 32 .. 128)
	{
		chars[i] = gl.glGenLists(1)
		gl.glNewList(chars[i], gl.GL_COMPILE)
			gl.glBegin(gl.GL_LINES)
				draw(toChar(i))
			gl.glEnd()
		gl.glEndList()
	}
}

function drawText(x: int, y: int, vararg)
{
	local len = sbuf.formatPos(0, vararg)

	gl.glPushMatrix()
		gl.glTranslatef(x, y, 0)

		for(i: 0 .. len)
		{
			drawChar(sbuf[i])
			gl.glTranslatef(24, 0, 0)
		}
	gl.glPopMatrix()
}

function drawCenterText(x: int, y: int, vararg)
{
	local str = format(vararg)
	drawText(x - #str * 12, y, str)
}

function drawRightText(x: int, y: int, vararg)
{
	local str = format(vararg)
	drawText(x - #str * 24, y, str)
}

local px, py

local function move(x, y)
	px, py = x, y

local function line(x, y)
{
	gl.glVertex2f(px, py)
	gl.glVertex2f(x, y)
	move(x, y)
}

draw = function draw(c: char)
{
	if(!(c.isLower() || c.isUpper() || c.isDigit() || c == ':' || c == '!'))
		c = ' '

	switch(c)
	{
		case 'A', 'a':
			move(0, 0)
			line(0, -10)
			line(8, -20)
			line(16, -10)
			line(16, 0)
			move(0, -10)
			line(16, -10)
			break

		case 'B', 'b':
			move(0, 0)
			line(0, -20)
			line(16, -10)
			line(0, -10)
			line(16, 0)
			line(0, 0)
			break

		case 'C', 'c':
			move(16, -20)
			line(0, -20)
			line(0, 0)
			line(16, 0)
			break

		case 'D', 'd':
			move(0, 0)
			line(0, -20)
			line(16, -10)
			line(16, 0)
			line(0, 0)
			break

		case 'E', 'e':
			move(16, -20)
			line(0, -20)
			line(0, 0)
			line(16, 0)
			move(0, -10)
			line(10, -10)
			break

		case 'F', 'f':
			move(16, -20)
			line(0, -20)
			line(0, 0)
			move(0, -10)
			line(10, -10)
			break

		case 'G', 'g':
			move(16, -20)
			line(0, -20)
			line(0, 0)
			line(16, 0)
			line(16, -10)
			line(10, -10)
			break

		case 'H', 'h':
			move(0, 0)
			line(0, -20)
			move(0, -10)
			line(16, -10)
			move(16, 0)
			line(16, -20)
			break

		case 'I', 'i':
			move(0, 0)
			line(16, 0)
			move(8, 0)
			line(8, -20)
			move(0, -20)
			line(16, -20)
			break

		case 'J', 'j':
			move(0, -20)
			line(16, -20)
			move(12, -20)
			line(12, 0)
			line(0, 0)
			line(0, -5)
			break

		case 'K', 'k':
			move(0, 0)
			line(0, -20)
			move(16, -20)
			line(0, -10)
			line(16, 0)
			break

		case 'L', 'l':
			move(0, -20)
			line(0, 0)
			line(16, 0)
			break

		case 'M', 'm':
			move(0, 0)
			line(0, -20)
			line(8, -10)
			line(16, -20)
			line(16, 0)
			break

		case 'N', 'n':
			move(0, 0)
			line(0, -20)
			line(16, 0)
			line(16, -20)
			break

		case 'O', 'o':
			move(0, 0)
			line(0, -20)
			line(16, -20)
			line(16, 0)
			line(0, 0)
			break

		case 'P', 'p':
			move(0, 0)
			line(0, -20)
			line(16, -20)
			line(16, -10)
			line(0, -10)
			break

		case 'Q', 'q':
			move(8, -10)
			line(16, 0)
			line(0, 0)
			line(0, -20)
			line(16, -20)
			line(16, 0)
			break

		case 'R', 'r':
			move(0, 0)
			line(0, -20)
			line(16, -20)
			line(16, -10)
			line(0, -10)
			line(16, 0)
			break

		case 'S', 's':
			move(16, -20)
			line(0, -20)
			line(0, -10)
			line(16, -10)
			line(16, 0)
			line(0, 0)
			break

		case 'T', 't':
			move(0, -20)
			line(16, -20)
			move(8, 0)
			line(8, -20)
			break

		case 'U', 'u':
			move(0, -20)
			line(0, 0)
			line(16, 0)
			line(16, -20)
			break

		case 'V', 'v':
			move(0, -20)
			line(0, -10)
			line(8, 0)
			line(16, -10)
			line(16, -20)
			break

		case 'W', 'w':
			move(0, -20)
			line(0, 0)
			line(8, -10)
			line(16, 0)
			line(16, -20)
			break

		case 'X', 'x':
			move(0, -20)
			line(16, 0)
			move(0, 0)
			line(16, -20)
			break

		case 'Y', 'y':
			move(0, -20)
			line(8, -10)
			line(8, 0)
			move(8, -10)
			line(16, -20)
			break

		case 'Z', 'z':
			move(0, -20)
			line(16, -20)
			line(0, 0)
			line(16, 0)
			break

		case '0':
			move(16, -20)
			line(0, 0)
			line(0, -20)
			line(16, -20)
			line(16, 0)
			line(0, 0)
			break

		case '1':
			move(0, 0)
			line(16, 0)
			move(8, 0)
			line(8, -20)
			line(4, -15)
			break

		case '2':
			move(0, -20)
			line(16, -20)
			line(16, -10)
			line(0, -10)
			line(0, 0)
			line(16, 0)
			break

		case '3':
			move(0, 0)
			line(16, 0)
			line(16, -20)
			line(0, -20)
			move(16, -10)
			line(0, -10)
			break

		case '4':
			move(16, 0)
			line(16, -20)
			line(0, -10)
			line(16, -10)
			break

		case '5':
			move(16, -20)
			line(0, -20)
			line(0, -10)
			line(16, -10)
			line(16, 0)
			line(0, 0)
			break

		case '6':
			move(16, -20)
			line(0, -20)
			line(0, 0)
			line(16, 0)
			line(16, -10)
			line(0, -10)
			break

		case '7':
			move(0, -20)
			line(16, -20)
			line(16, 0)
			break

		case '8':
			move(0, 0)
			line(0, -20)
			line(16, -20)
			line(16, 0)
			line(0, 0)
			move(0, -10)
			line(16, -10)
			break

		case '9':
			move(0, 0)
			line(16, 0)
			line(16, -20)
			line(0, -20)
			line(0, -10)
			line(16, -10)
			break

		case ':':
			move(8, -18)
			line(8, -14)
			move(8, -6)
			line(8, -2)
			break

		case '!':
			move(8, -20)
			line(8, -8)
			move(8, -3)
			line(8, 0)
			break

		case ' ': break
	}
}