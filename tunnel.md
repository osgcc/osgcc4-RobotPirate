module tunnel

import gl
import sdl: event, key
import math: sin, cos, abs

import text

local ToRad = math.pi / 180
local ToDeg = 180 / math.pi

import config

/*
As level progresses:
	- Randomly spawn mines
		- In patterns?
	- Randomly spawn enemies
	- Mine-dodging and dogfighting sections (where you do one to the exclusion of the other)
	- Spawn boss every some amount of distance.. and let the player kill them before resuming normal play
		- Increments difficulty level

Enemies drop items randomly
	- Shield?
*/

function main()
{
	setup()
	scope(exit)
		sdl.quit()

	while(mainScreenTurnOn())
	{
		intro()
		loop()
	}
}

function FreeList(T: class)
{
	T._freelist_ = null
	T.next = null
	
	if(hasMethod(T, "initialize"))
	{
		T.alloc = function alloc(this: class, vararg)
		{
			local n

			if(:_freelist_)
			{
				n = :_freelist_
				:_freelist_ = n.next
			}
			else
				n = this()

			n.initialize(vararg)
			return n
		}
	}
	else
	{
		T.alloc = function alloc(this: class)
		{
			local n

			if(:_freelist_)
			{
				n = :_freelist_
				:_freelist_ = n.next
			}
			else
				n = this()

			return n
		}
	}

	T.free = function free(this: instance)
	{
		:next = :_freelist_
		:super._freelist_ = this
	}

	return T
}

@FreeList
class Mine
{
	ang
	z
	prev
	health

	function initialize(ang: float, z: float)
	{
		:ang = ang
		:z = z
		:health = 2
	}
}

@FreeList
class Bullet
{
	ang
	z
	dir
	prev
	damage

	function initialize(ang: float, z: float, dir: float, damage: int)
	{
		:ang = ang
		:z = z
		:dir = dir * ToRad
		:damage = damage
	}
}

@FreeList
class Enemy
{
	ang
	z
	health
	weaponDelayer
	desc
	dAng
	delay
	dZ

	function initialize(ang: float, z: float, desc: table)
	{
		:ang = ang
		:z = z
		:desc = desc
		:weaponDelayer = desc.fireRate
		:health = desc.health
		:dAng = 0
		:delay = 0
		:dZ = 0
	}
	
	function update()
	{
		:desc.update(with this)
	}
}

@FreeList
class Item
{
	ang
	z
	desc
	
	function initialize(ang: float, z: float, desc: table)
	{
		:ang = ang
		:z = z
		:desc = desc
	}
	
	function collect()
	{
		:desc.collect(with this)
	}
}

// ===================================================================================
// State
// ===================================================================================

local projMat2D = Vector(gl.GLfloat, 16)
local projMat3D = Vector(gl.GLfloat, 16)
local fps = 0
local drawDebugText = false

local gameOver = false
local quitting = false
local keys = array.new(512, false)
local keysHit = array.new(512, false)

local playerMoveSpeed = 12
local playerMoveMult = 0.15
local playerWeaponChangeDelay = 20
local playerMaxHealth = 30
local playerModel
local playerAng
local dPlayerAng
local playerHealth
local playerWeaponLevel
local playerWeapon
local playerScore
local playerLives
local playerFiringRailgun
local playerDeathCounter

local playerAngLag = array.new(9, 0)
local lagNewPtr
local lagOldPtr
local camAng

local levelSegments = 30
local levelSegmentLen = 8
local levelSpeed = 1.1
local levelModel
local levelOffs
global levelZ
local levelMode
local nextLevelChange
local levelChangeLength = 600
local normalMineFreq = 75
local normalEnemyFreq = 100
local minefieldMineFreq = 7
local MODE_NORMAL = 0
local MODE_MINEFIELD = 1
local MODE_DOGFIGHT = 2
local MODE_BOSS = 3
local MODE_DEAD = 4

local mineDamageAmount = 5
local mines = Mine() // dummy sentry node
local mineModel
local mineAng

local bulletSpeed = 2
local playerBulletDelay = 14
local playerBullets = Bullet() // dummy sentry node
local playerBulletDelayer
local bulletModel

local enemyBullets = Bullet() // dummy sentry node

local enemies = Enemy() // dummy sentry node

local items = Item() // dummy sentry node
local itemFlashCounter = 0

namespace Scores
{
	MINE = 15
	DERP = 30
	QUICK = 60
	HEAVY = 100
	BOSS = 1000
	ITEM = 50
}

global weaponDescs =
[
	{
		level = 0
		name = "shooter1"
		fireRate = 14
		damage = 1

		function fire()
		{
			insertIntoList(Bullet.alloc(playerAng, -30.0, 0.0, :damage), playerBullets)
		}
	}

	{
		level = 1
		name = "shooter2"
		fireRate = 12
		damage = 2

		function fire()
		{
			insertIntoList(Bullet.alloc(wrapAngle(playerAng + 3), -30.0, 0.0, :damage), playerBullets)
			insertIntoList(Bullet.alloc(wrapAngle(playerAng - 3), -30.0, 0.0, :damage), playerBullets)
		}
	}

	{
		level = 2
		name = "shooter3"
		fireRate = 18
		damage = 2

		function fire()
		{
			insertIntoList(Bullet.alloc(playerAng, -30.0, 0.0, :damage), playerBullets)
			insertIntoList(Bullet.alloc(wrapAngle(playerAng + 3), -30.0, 15.0, :damage), playerBullets)
			insertIntoList(Bullet.alloc(wrapAngle(playerAng - 3), -30.0, -15.0, :damage), playerBullets)
		}
	}
	
	{
		level = 3
		name = "railgun"
		fireRate = 60
		damage = 13

		function fire()
		{
			playerFiringRailgun = 4
		}
	}
]

// Enemy types:
// 	- Derp. Moves slightly slower than the level itself, shoots ahead every once in a while, eventually moves off near horizon. Little health.
// 	- Quick. Moves forward/back and around a little way ahead of the player. Shoots at player more often than derp, but not too quick. Eventually moves off near horizon. Medium health.
// 	- Heavy. Not very mobile, but hovers ahead of player. Fires a lot (maybe spread shot). Must be killed. High health.
// 	- Boss. Not too mobile, but must be hit in weak spots. Fires a lot. Must be killed. Very high health.

local enemyDescs =
[
	{
		name = "derp"
		weapon = 0
		fireRate = 120
		health = 3
		model = 0
		score = Scores.DERP

		function update()
		{
			:z += 0.15 * levelSpeed

			:dAng = math.rand(3) - 1
			:ang = wrapAngle(:ang + :dAng)
			
			:weaponDelayer--
			
			if(:weaponDelayer <= 0)
			{
				:weaponDelayer = :desc.fireRate
				insertIntoList(Bullet.alloc(:ang, :z, 0.0, weaponDescs[:desc.weapon].damage), enemyBullets)
			}
		}
	}
	
	{
		name = "quick"
		weapon = 0
		fireRate = 60
		health = 4
		model = 0
		score = Scores.QUICK

		function update()
		{
			if(:z < -60)
			{
				:dZ = levelSpeed
				:dAng = 0
			}
			else
			{
				:delay--

				if(:delay <= 0)
				{
					:delay = 20

					if(:dZ == levelSpeed)
					{
						:dZ = 0.2 * levelSpeed
						:dAng = 1
					}
					else
					{
						:dZ = -:dZ
						:dAng = math.rand(3) - 1
					}
				}
			}

			:z += :dZ
			:ang = wrapAngle(:ang + :dAng)

			:weaponDelayer--
			
			if(:weaponDelayer <= 0)
			{
				:weaponDelayer = :desc.fireRate
				insertIntoList(Bullet.alloc(:ang, :z, 0.0, weaponDescs[:desc.weapon].damage), enemyBullets)
			}
		}
	}
	
	{
		name = "heavy"
		weapon = 2
		fireRate = 60
		health = 10
		model = 0
		score = Scores.HEAVY
		
		function update()
		{
			if(:z < -70)
			{
				:dZ = levelSpeed
				:dAng = 0
			}
			else
			{
				:delay--
				
				if(:delay <= 0)
				{
					:delay = 20
					
					if(:dZ == levelSpeed)
						:dZ = 0

					:dAng = math.frand(0.5) - 0.25
				}
			}
			
			:z += :dZ
			:ang = wrapAngle(:ang + :dAng)
			
			:weaponDelayer--
			
			if(:weaponDelayer <= 0)
			{
				:weaponDelayer = :desc.fireRate
// 				insertIntoList(Bullet.alloc(:ang, :z, 0.0, weaponDescs[:desc.weapon].damage), enemyBullets)
				insertIntoList(Bullet.alloc(wrapAngle(:ang + 5), :z, 10.0, weaponDescs[:desc.weapon].damage), enemyBullets)
				insertIntoList(Bullet.alloc(wrapAngle(:ang - 5), :z, -10.0, weaponDescs[:desc.weapon].damage), enemyBullets)
			}
		}
	}
]

local itemDescs =
[
	{
		name = "health"
		model = 0

		function collect()
		{
			healPlayer(playerMaxHealth / 5)
		}
	}
	
	{
		name = "weapon"
		model = 0
		
		function collect()
		{
			playerWeaponLevel = clamp(playerWeaponLevel + 1, 0, 3)
		}
	}
	
	{
		name = "extralife"
		model = 0
		
		function collect()
		{
			playerLives++
		}
	}
]

// ===================================================================================
// Code
// ===================================================================================

function mainScreenTurnOn()
{
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadMatrixf(projMat2D)
	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glDisable(gl.GL_LIGHTING)

	resetState()

	local ret

	while(true)
	{
		event.poll()

		if(keys[key.escape])
		{
			ret = false
			break
		}

		if(keysHit[key.space])
		{
			ret = true
			break
		}

		gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
			gl.glLoadIdentity()

			gl.glColor3f(1, 1, 0)

			gl.glPushMatrix()
				gl.glScalef(4, 4, 1)
				text.drawCenterText(config.winCenterX / 4, 30, "ROBOT")
				text.drawCenterText(config.winCenterX / 4, 60, "PIRATE")
			gl.glPopMatrix()

			gl.glColor3f(1, 1, 1)
			text.drawCenterText(config.winCenterX, 300, "A GAME FOR THE 4TH ANNUAL OSGCC")
			text.drawCenterText(config.winCenterX, 325, "BY JARRETT BILLINGSLEY")
			text.drawCenterText(config.winCenterX, 400, "PRESS SPACE TO PLAY")
			text.drawCenterText(config.winCenterX, 425, "OR ESCAPE TO QUIT")
		sdl.gl.swapBuffers()
	}

	gl.glColor3f(1, 1, 1)
	return ret
}

function intro()
{
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadMatrixf(projMat2D)
	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glDisable(gl.GL_LIGHTING)

	resetState()

	while(true)
	{
		event.poll()

		if(keysHit[key.space])
			break

		gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
			gl.glLoadIdentity()

			gl.glColor3f(1, 1, 0)
			text.drawCenterText(config.winCenterX, 45, "YOU ARE ROBOT PIRATE")
			gl.glColor3f(1, 1, 1)
			text.drawText(5, 95 , "THE YEAR IS 20XX. EARTH HAS BEEN")
			text.drawText(5, 120, "OPENED TO INTERGALACTIC TRADE.")
			text.drawText(5, 145, "BUT SECURITY IS LOOSE AND PIRATES")
			text.drawText(5, 170, "ARE A COMMON SIGHT.")
			text.drawText(5, 220, "YOU ARE A SYNTHETIC BEING CREATED")
			text.drawText(5, 245, "BY A CRIME SYNDICATE. HOWEVER YOU")
			text.drawText(5, 270, "HAVE BECOME TRAPPED DEEP INSIDE A")
			text.drawText(5, 295, "LARGE SPACE STATION WHILE ON A")
			text.drawText(5, 320, "MISSION. NOW YOU MUST FIGHT YOUR")
			text.drawText(5, 345, "WAY OUT AGAINST THE OTHER ROBOTIC")
			text.drawText(5, 370, "GUARDS. GOOD LUCK!")
			gl.glColor3f(1, 1, 0)
			text.drawCenterText(config.winCenterX, 420, "CONTROLS")
			gl.glColor3f(1, 1, 1)
			text.drawText(5, 470, "LEFT AND RIGHT MOVE. SPACE FIRES.")
			text.drawText(5, 495, "ZXCV SWITCH WEAPONS. ESC EXITS.")
			text.drawText(5, 545, "PRESS SPACE TO START!")
		sdl.gl.swapBuffers()
	}

	gl.glColor3f(1, 1, 1)
}

function gameOverMan()
{
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadMatrixf(projMat2D)
	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glDisable(gl.GL_LIGHTING)

	resetState()

	while(true)
	{
		event.poll()

		if(keysHit[key.space] || keysHit[key.escape])
			break

		gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)

			gl.glLoadIdentity()

			gl.glColor3f(1, 0, 0)

			gl.glPushMatrix()
				gl.glScalef(4, 4, 1)
				text.drawCenterText(config.winCenterX / 4, 25, "GAME")
				text.drawCenterText(config.winCenterX / 4, 50, "OVER")
			gl.glPopMatrix()

			gl.glColor3f(1, 1, 1)

			text.drawText(5, 250, "              XXXXX           ")
			text.drawText(5, 275, "            XX     XX         ")
			text.drawText(5, 300, "     XXX   X         X   XXX  ")
			text.drawText(5, 325, "    XXXX   X XX   XX X   XXXX ")
			text.drawText(5, 350, "      XXX X  XX   XX  X XXX   ")
			text.drawText(5, 375, "        XX X   X X   X XX     ")
			text.drawText(5, 400, "         X  X       X  X      ")
			text.drawText(5, 425, "            X X X X X         ")
			text.drawText(5, 450, "             X X X X          ")
			text.drawText(5, 475, "              XXXXX           ")
			text.drawText(5, 500, "            X       X         ")
			text.drawText(5, 525, "          XXXX     XXXX       ")
			text.drawText(5, 550, "         XXXX       XXXX      ")
			text.drawText(5, 575, "          XX         XX       ")
		sdl.gl.swapBuffers()
	}

	gl.glColor3f(1, 1, 1)
}


function resetState()
{
	gameOver = false
	quitting = false
	keys.fill(false)
	keysHit.fill(false)
	playerAng = 0.0
	dPlayerAng = 0
	playerHealth = playerMaxHealth
	playerFiringRailgun = 0
	playerBulletDelayer = 0
	playerWeaponLevel = 0
	playerWeapon = weaponDescs[0]
	playerScore = 0
	playerLives = 3
	playerAngLag.fill(0)
	lagNewPtr = #playerAngLag - 1
	lagOldPtr = 0
	camAng = 0.0
	mineAng = 0.0
	levelOffs = 0.0
	levelZ = 0.0
	levelMode = MODE_NORMAL
	nextLevelChange = 0

	emptyList(mines)
	emptyList(playerBullets)
	emptyList(enemyBullets)
	emptyList(enemyBullets)
	emptyList(items)
}

// Game setup
function setup()
{
	setupGraphics()
	setupResources()
	setupEvents()
	resetState()
}

// Main game loop.
function loop()
{
	resetState()
	local t = time.microTime()
	local frames = 0

	while(!gameOver && !quitting)
	{
		if(levelMode != MODE_DEAD)
		{
			doInput()
			updatePlayer()
		}

		updateCamera()
		updateLevel()
		updateEnemies()
		updateItems()
		drawGraphics()

		frames++
		local dt = time.microTime() - t
		
		if(dt >= 1_000_000)
		{
			t = time.microTime()
			fps = frames / (dt / 1_000_000.0)
			frames = 0
		}
	}
	
	if(gameOver)
		gameOverMan()
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
	gl.glGetFloatv(gl.GL_PROJECTION_MATRIX, projMat3D)

	gl.glLoadIdentity()
	gl.gluOrtho2D(0, config.winWidth, 0, config.winHeight);
	gl.glScalef(1, -1, 1);
	gl.glTranslatef(0.5, -config.winHeight + 0.5, 0);
	gl.glGetFloatv(gl.GL_PROJECTION_MATRIX, projMat2D)

	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_LINE);
	gl.glLightModelfv(gl.GL_LIGHT_MODEL_AMBIENT, float4(0, 0, 0, 1))

	gl.glEnable(gl.GL_LIGHT0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_SPOT_DIRECTION, float4(0, -1, 0, 0))
// 	gl.glLightf(gl.GL_LIGHT0, gl.GL_LINEAR_ATTENUATION, 0.001);
	gl.glLightf(gl.GL_LIGHT0, gl.GL_QUADRATIC_ATTENUATION, 0.1);
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -6, -26, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_DIFFUSE, float4(0, 10, 0, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_AMBIENT, float4(0, 10, 0, 1))

	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)

	text.setup()
}

// Load resources
function setupResources()
{
	levelModel = makeTunnelPiece(20)

	playerModel = loadModel("models/ship.obj")
	mineModel = loadModel("models/obstacle.obj")
	bulletModel = loadModel("models/bullet.obj")

	enemyDescs[0].model = loadModel("models/derp.obj")
	enemyDescs[1].model = loadModel("models/quick.obj")
	enemyDescs[2].model = loadModel("models/heavy.obj")

	itemDescs[0].model = loadModel("models/health.obj")
	itemDescs[1].model = loadModel("models/weapon.obj")
	itemDescs[2].model = loadModel("models/extralife.obj")
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
		dPlayerAng -= 1
	else if(keys[key.right] && !keys[key.left])
		dPlayerAng += 1
	else
	{
		if(dPlayerAng > 0)
			dPlayerAng -= 1
		else if(dPlayerAng < 0)
			dPlayerAng += 1
	}

	local weaponChange = playerWeapon.level

	if(keysHit[key.z])
		weaponChange = 0
	else if(keysHit[key.x] && playerWeapon.level != 1)
		weaponChange = 1
	else if(keysHit[key.c] && playerWeapon.level != 2)
		weaponChange = 2
	else if(keysHit[key.v] && playerWeapon.level != 3)
		weaponChange = 3

	if(weaponChange <= playerWeaponLevel && weaponChange != playerWeapon.level)
		changeWeapon(weaponChange)

	if(playerBulletDelayer == 0 && keys[key.space])
	{
		playerBulletDelayer = playerWeapon.fireRate
		playerWeapon.fire()
	}
	
	drawDebugText = keys[key.backquote]
}

function changeWeapon(idx: int)
{
	playerWeapon = weaponDescs[idx]
	playerBulletDelayer = math.max(playerWeaponChangeDelay, playerWeapon.fireRate)
}

// Player
function updatePlayer()
{
	dPlayerAng = clamp(dPlayerAng, -playerMoveSpeed, playerMoveSpeed)
	playerAng = wrapAngle(playerAng + dPlayerAng * playerMoveMult)

	if(playerBulletDelayer > 0)
		playerBulletDelayer--

	for(local bul = playerBullets; bul && bul.next !is null; bul = bul.next)
	{
		local b = bul.next
		
		local dAng, dZ = sin(b.dir) * bulletSpeed, cos(b.dir) * bulletSpeed

		b.ang = wrapAngle(b.ang + dAng)
		b.z -= dZ

		if(b.z < -180)
		{
			removeFromList(b)
			b.free()
			continue
		}
		
		local collided = false

		// Check for collision with enemies
		for(local enemy = enemies.next; enemy !is null; enemy = enemy.next)
		{
			if(abs(b.z - enemy.z) < 2 && angDiff(b.ang, enemy.ang) < 7)
			{
				damageEnemy(enemy, b.damage)
				removeFromList(b)
				b.free()
				collided = true
				break
			}
		}
		
		if(collided)
			continue

		// Check for collision with mines
		for(local mine = mines.next; mine !is null; mine = mine.next)
		{
			if(abs(b.z - mine.z) < 2 && angDiff(b.ang, mine.ang) < 7)
			{
				mine.health -= b.damage
				
				if(mine.health <= 0)
				{
					playerScore += Scores.MINE
					removeFromList(mine)
					mine.free()
				}

				removeFromList(b)
				b.free()
				break
			}
		}
	}

	if(playerFiringRailgun)
	{
		// Check for collision with mines
		for(local mine = mines; mine && mine.next !is null; mine = mine.next)
		{
			local m = mine.next

			if(m.z < -30 && angDiff(playerAng, m.ang) < 7)
			{
				playerScore += Scores.MINE
				removeFromList(m)
				m.free()
			}
		}
		
		// Check for collision with enemies
		for(local enemy = enemies; enemy && enemy.next !is null; enemy = enemy.next)
		{
			local e = enemy.next

			if(e.z < -30 && angDiff(playerAng, e.ang) < 7)
				damageEnemy(e, weaponDescs[3].damage)
		}

		playerFiringRailgun--
	}
}

function damageEnemy(enemy, amt)
{
	enemy.health -= amt

	if(enemy.health <= 0)
	{
		playerScore += enemy.desc.score
		maybeSpawnItem(enemy.ang, enemy.z)
		removeFromList(enemy)
		enemy.free()
	}
}

// Camera
function updateCamera()
{
	camAng = wrapAngle(camAng + playerAngLag[lagOldPtr] * playerMoveMult)
	playerAngLag[lagNewPtr] = dPlayerAng
	lagNewPtr++
	if(lagNewPtr == #playerAngLag)
		lagNewPtr = 0
	lagOldPtr++
	if(lagOldPtr == #playerAngLag)
		lagOldPtr = 0
}

// Level stuff
function updateLevel()
{
	levelZ += levelSpeed

	switch(levelMode)
	{
		case MODE_NORMAL:
			if(levelZ % normalMineFreq < 1)
				insertIntoList(Mine.alloc(math.frand(360.0), -240.0), mines)

			if(levelZ % normalEnemyFreq < 1)
			{
				local r = math.frand(1.0)
				local type

				if(r < 0.7)
					type = 0
				else if(r < 0.95)
					type = 1
				else
					type = 2

				insertIntoList(Enemy.alloc(math.frand(360.0), -120.0, enemyDescs[type]), enemies)
			}

			if(levelZ >= nextLevelChange)
			{
				levelMode = math.rand(3)
				nextLevelChange = levelZ + levelChangeLength
			}
			break

		case MODE_MINEFIELD:
			if(levelZ % minefieldMineFreq < 1)
				insertIntoList(Mine.alloc(math.frand(360.0), -240.0), mines)
			
			if(levelZ >= nextLevelChange)
			{
				levelMode = math.rand(2)

				if(levelMode > 0)
					levelMode++
					
				nextLevelChange = levelZ + levelChangeLength
			}
			break

		case MODE_DOGFIGHT:
			// TODO: this
			levelMode = MODE_NORMAL
			nextLevelChange = levelZ
			break

		case MODE_BOSS:
			// TODO: this
			assert(false)
			break
			
		case MODE_DEAD:
			playerDeathCounter--

			if(playerDeathCounter <= 0)
			{
				if(playerLives == 0)
				{
					gameOver = true
				}
				else
				{
					emptyList(playerBullets)
					emptyList(enemyBullets)
					emptyList(mines)
					emptyList(enemies)
					levelMode = MODE_NORMAL
					healPlayer(playerMaxHealth)
				}
			}
			break
	}

	levelOffs += levelSpeed

	if(levelOffs > levelSegmentLen)
		levelOffs -= levelSegmentLen

	mineAng = wrapAngle(mineAng + 5)

	for(local mine = mines; mine && mine.next !is null; mine = mine.next)
	{
		local m = mine.next

		m.z += levelSpeed
		
		if(abs(m.z + 30) < 2 && angDiff(m.ang, playerAng) < 15)
		{
			damagePlayer(mineDamageAmount, true)
			removeFromList(m)
			m.free()
			continue
		}

		if(m.z > 0)
		{
			removeFromList(m)
			m.free()
			continue
		}
	}
}

function updateEnemies()
{
	for(local en = enemies; en && en.next; en = en.next)
	{
		local e = en.next

		e.update()
		
		if(e.z > 0)
		{
			removeFromList(e)
			e.free()
		}
	}
	
	for(local bul = enemyBullets; bul && bul.next; bul = bul.next)
	{
		local b = bul.next
		local dAng, dZ = sin(b.dir) * bulletSpeed * 0.6, cos(b.dir) * bulletSpeed * 0.6

		b.ang = wrapAngle(b.ang + dAng)
		b.z += dZ

		if(b.z > 0)
		{
			removeFromList(b)
			b.free()
			continue
		}

		// Check for collision with player
		if(abs(b.z + 30) < 2 && angDiff(b.ang, playerAng) < 10)
		{
			damagePlayer(b.damage)
			removeFromList(b)
			b.free()
			continue
		}

		// Check for collision with mines
		for(local mine = mines.next; mine !is null; mine = mine.next)
		{
			if(abs(b.z - mine.z) < 2 && angDiff(b.ang, mine.ang) < 7)
			{
				removeFromList(mine)
				mine.free()
				removeFromList(b)
				b.free()
				break
			}
		}
	}
}

function updateItems()
{
	itemFlashCounter = (itemFlashCounter + 1) & 0b1111

	for(local item = items; item && item.next; item = item.next)
	{
		local i = item.next

		i.z += 0.4 * levelSpeed

		if(i.z > 0)
		{
			removeFromList(i)
			i.free()
			continue
		}

		if(abs(i.z + 30) < 2 && angDiff(i.ang, playerAng) < 15)
		{
			playerScore += Scores.ITEM
			i.collect()
			removeFromList(i)
			i.free()
		}
	}
}

// Graphics
function drawGraphics()
{
	gl.glClear(gl.GL_COLOR_BUFFER_BIT | gl.GL_DEPTH_BUFFER_BIT)
		draw3D()
		draw2D()
	sdl.gl.swapBuffers()
}

function draw3D()
{
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadMatrixf(projMat3D)
	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glEnable(gl.GL_LIGHTING)

	gl.glLoadIdentity()
	gl.glRotatef(-camAng, 0, 0, 1)

	drawLevel()
	drawMines()
	
	if(levelMode != MODE_DEAD)
		drawPlayer()
		
	drawPlayerBullets()
	drawEnemies()
	drawEnemyBullets()
	drawItems()
}

function drawLevel()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.05, 0.05, 0.05, 1)

	gl.glPushMatrix()
		gl.glTranslatef(0, 0, levelOffs)
		for(i: 0 .. levelSegments)
		{
			gl.glTranslatef(0, 0, -levelSegmentLen)
			gl.glCallList(levelModel)
		}
	gl.glPopMatrix()
}

function drawMines()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 0, 0, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.8, 0, 0, 1)

	for(local mine = mines.next; mine !is null; mine = mine.next)
	{
		gl.glPushMatrix()
			gl.glRotatef(mine.ang, 0, 0, 1)
			gl.glTranslatef(0, -9, mine.z)
			gl.glRotatef(mineAng, 0, 1, 0)
			gl.glCallList(mineModel)
		gl.glPopMatrix()
	}
}

function drawPlayerBullets()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(0, 0, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0, 0, 1, 1)

	for(local bullet = playerBullets.next; bullet !is null; bullet = bullet.next)
	{
		gl.glPushMatrix()
			gl.glRotatef(bullet.ang, 0, 0, 1)
			gl.glTranslatef(0, -9, bullet.z)
			gl.glCallList(bulletModel)
		gl.glPopMatrix()
	}

	if(playerFiringRailgun)
	{
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 0, 1)
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(1, 1, 0, 1)
		
		gl.glPushMatrix()
			gl.glRotatef(playerAng, 0, 0, 1)
			gl.glTranslatef(0, -9, -30)
			gl.glBegin(gl.GL_LINE_STRIP)
				for(i: 0 .. 240, 20)
				{
					gl.glVertex3f(1, 0, -i)
					gl.glVertex3f(-1, 0, -i - 10)
				}
			gl.glEnd()
		gl.glPopMatrix()
	}
}

function drawPlayer()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.8, 0.8, 0.8, 1)

	gl.glPushMatrix()
		gl.glRotatef(playerAng, 0, 0, 1)
		gl.glLightfv(gl.GL_LIGHT0, gl.GL_POSITION, float4(0, -8, -23, 1))
		gl.glTranslatef(0, -9, -30)
		gl.glCallList(playerModel)
	gl.glPopMatrix()
}

function drawEnemies()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(0.8, 0.8, 0.8, 1)

	for(local e = enemies.next; e !is null; e = e.next)
	{
		gl.glPushMatrix()
			gl.glRotatef(e.ang, 0, 0, 1)
			gl.glTranslatef(0, -9, e.z)
			gl.glRotatef(180, 0, 1, 0)
			gl.glCallList(e.desc.model)
		gl.glPopMatrix()
	}
}

function drawEnemyBullets()
{
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 0, 1, 1)
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(1, 0, 1, 1)

	for(local bullet = enemyBullets.next; bullet !is null; bullet = bullet.next)
	{
		gl.glPushMatrix()
			gl.glRotatef(bullet.ang, 0, 0, 1)
			gl.glTranslatef(0, -9, bullet.z)
			gl.glCallList(bulletModel)
		gl.glPopMatrix()
	}
}

function drawItems()
{
	if(itemFlashCounter >= 8)
	{
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(1, 1, 1, 1)
	}
	else
	{
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 0, 0, 1)
		gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_EMISSION, float4(1, 0, 0, 1)
	}

	for(local item = items.next; item !is null; item = item.next)
	{
		gl.glPushMatrix()
			gl.glRotatef(item.ang, 0, 0, 1)
			gl.glTranslatef(0, -9, item.z)
			gl.glCallList(item.desc.model)
		gl.glPopMatrix()
	}
}

function draw2D()
{
	gl.glMatrixMode(gl.GL_PROJECTION)
	gl.glLoadMatrixf(projMat2D)
	gl.glMatrixMode(gl.GL_MODELVIEW)
	gl.glDisable(gl.GL_LIGHTING)
	gl.glLoadIdentity()
	gl.glMaterialfv$ gl.GL_FRONT_AND_BACK, gl.GL_AMBIENT_AND_DIFFUSE, float4(1, 1, 1, 1)

	if(drawDebugText)
	{
		text.drawText(0, 20, "FPS:{} MEM:{}", toInt(fps), gc.allocated())
		text.drawText(0, 45, "MODE:{}", levelMode)
		text.drawText(0, 70, "{}:{}", levelZ, nextLevelChange)
	}

	text.drawText(10, config.winHeight - 10, "LIVES:{}", playerLives)
	text.drawRightText(config.winWidth - 10, config.winHeight - 10, "{}", playerScore)

	drawHealthBar()
}

function drawHealthBar()
{
	gl.glPushMatrix()
		gl.glTranslatef(config.winCenterX, config.winHeight - 30, 0)
		gl.glBegin(gl.GL_LINE_STRIP)
			gl.glVertex2f(-80, 10)
			gl.glVertex2f(80, 10)
			gl.glVertex2f(80, -10)
			gl.glVertex2f(-80, -10)
			gl.glVertex2f(-80, 10)
		gl.glEnd()

		local temp = playerHealth / toFloat(playerMaxHealth)
		gl.glColor3f(1 - temp, temp, 0)
		gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_FILL);

		gl.glScalef(temp, 1, 1)
		gl.glBegin(gl.GL_QUADS)
			gl.glVertex2f(-80, 10)
			gl.glVertex2f(80, 10)
			gl.glVertex2f(80, -10)
			gl.glVertex2f(-80, -10)
		gl.glEnd()

		gl.glColor3f(1, 1, 1)
		gl.glPolygonMode(gl.GL_FRONT_AND_BACK, gl.GL_LINE);

	gl.glPopMatrix()
}

function damagePlayer(amt: int, explosive: bool = false)
{
	if(levelMode == MODE_DEAD)
		return

	playerHealth = clamp(playerHealth - amt, 0, playerMaxHealth)
	local temp = playerHealth / (playerMaxHealth / 10.0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_DIFFUSE, float4(10 - temp, temp, 0, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_AMBIENT, float4(10 - temp, temp, 0, 1))

	if(playerHealth == 0)
	{
		levelMode = MODE_DEAD
		playerLives--
		playerDeathCounter = 180
	}
}

function healPlayer(amt: int)
{
	playerHealth = clamp(playerHealth + amt, 0, playerMaxHealth)
	local temp = playerHealth / (playerMaxHealth / 10.0)
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_DIFFUSE, float4(10 - temp, temp, 0, 1))
	gl.glLightfv(gl.GL_LIGHT0, gl.GL_AMBIENT, float4(10 - temp, temp, 0, 1))
}

function maybeSpawnItem(ang: float, z: float)
{
	local r = math.frand(1.0)

	if(r < 0.75)
		return

	r = math.frand(1.0)
	local type

	if(playerHealth <= playerMaxHealth / 2)
	{
		// health 50%
		// weapon upgrades 49%
		// extra life 1%

		if(r < 0.5)
			type = 0
		else if(r < 0.99)
			type = 1
		else
			type = 2
	}
	else
	{
		// health 20%
		// weapon upgrades 78%
		// extra life 2%

		if(r < 0.2)
			type = 0
		else if(r < 0.98)
			type = 1
		else
			type = 2
	}

	insertIntoList(Item.alloc(ang, z, itemDescs[type]), items);
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
	else if(v >= max)
	{
		local d = max - min
		while(v >= max)
			v -= d
	}

	return v
}

// Keeps v in the range [0, 360).
function wrapAngle(v)
{
	if(v < 0)
	{
		while(v < 0)
			v += 360
	}
	else if(v >= 360)
	{
		while(v >= 360)
			v -= 360
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

// Inserts the object o at the head of the doubly linked list l.
function insertIntoList(o, l)
{
	o.next = l.next
	if(o.next)
		o.next.prev = o
	o.prev = l
	l.next = o
}

// Removes the object o from its list.
function removeFromList(o)
{
	o.prev.next = o.next
	if(o.next)
		o.next.prev = o.prev
}

// Empties out a list
function emptyList(l)
{
	while(l.next)
	{
		local o = l.next
		removeFromList(o)
		o.free()
	}
}