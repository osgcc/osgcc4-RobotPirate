module game

import sdl: event, key

function loop()
{
	local quitting = false
	event.setHandler$ event.quit, \{ quitting = true }

	local keys = array.new(512, false)
	event.setHandler$ event.key, \pressed, sym, mod { keys[sym] = pressed }

	while(!quitting)
	{
		
	}
}