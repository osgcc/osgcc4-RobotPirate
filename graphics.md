module graphics

import sdl
import gl

import config

function init()
{
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

	gl.load()
	gl.glViewport(0, 0, config.winWidth, config.winHeight)
	gl.glShadeModel(gl.GL_FLAT)
	gl.glClearColor(0, 0, 0, 1)
	gl.glClearDepth(1)
	gl.glEnable(gl.GL_CULL_FACE)
	gl.glEnable(gl.GL_DEPTH_TEST)
// 	gl.glEnable(gl.GL_TEXTURE_2D)
	gl.glEnable(gl.GL_BLEND)
	gl.glEnable(gl.GL_NORMALIZE)
	gl.glBlendFunc(gl.GL_SRC_ALPHA, gl.GL_ONE_MINUS_SRC_ALPHA)
	gl.glEnable(gl.GL_LIGHTING)
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadIdentity()
	gl.gluPerspective(45, toFloat(config.winWidth) / config.winHeight, 3, 1000)
	gl.glMatrixMode(gl.GL_MODELVIEW)
}

function uninit()
{
	sdl.quit()
}