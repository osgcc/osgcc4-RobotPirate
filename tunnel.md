module tunnel

import graphics
import game

function main()
{
	graphics.init()

	scope(exit)
		graphics.uninit()
		
	game.loop()
}