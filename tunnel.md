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

/*
- player/camera/movement
- obstacles
- bullets
- enemies
- DERP
*/