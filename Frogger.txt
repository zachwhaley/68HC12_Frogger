*================================================*
*================== Frogger =====================*
*================================================*
*===== Copyright (c) 2010 by Zachary Whaley =====*
*================================================*
*============= ALL RIGHTS RESERVED ==============*
*================================================*
*===============December 6, 2010 ================*
*================================================*
*== Author: Zach Whaley and Christopher Garner ==*
*================================================*

*=========================================================================*
*=                                                                       =*
*= This program is free software: you can redistribute it and/or modify  =*
*= it under the terms of the GNU General Public License as published by  =*
*= the Free Software Foundation, either version 3 of the License, or     =*
*= (at your option) any later version.                                   =*
*=                                                                       =*  
*= This program is distributed in the hope that it will be useful,       =*
*= but WITHOUT ANY WARRANTY; without even the implied warranty of        =*
*= MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the         =*
*= GNU General Public License for more details.                          =*
*=                                                                       =*
*= You should have received a copy of the GNU General Public License     =*
*= along with this program.  If not, see <http://www.gnu.org/licenses/>. =*
*=                                                                       =*
*=========================================================================*


*=============
*= Registers =

*=============

**SPI registers**
*Controls the spi interrupt using SPIE*
*Interrupts everytime SPIF is set*
spocr1	equ	$08D0	;SPI control register 1
sposr	equ	$08D3	;SPI status register, Controls SPI interrupt request with SPIF
ports	equ	$08D6	;Pads are connected to ports
ddrs	equ	$08D7	;Port S data direction
spobr	equ	$08d2	;SPI baud rate
spodr	equ	$08d5	;SPI data register

portt	equ	$08AE	;Used for slave select of pad 2
ddrt	equ	$08AF

**RTI registers**
rtictl	equ	$0814	;controls real-time interrupts
rtiflg	equ	$0815	;RTI flag. set to 1 at end of period

**LCD Registers**
portg	equ	$0828	;Data bits for LCD
porth	equ	$0829	;Command bits for LCD
ddrg	equ	$082A	
ddrh	equ	$082B

**Game State Bits**
TRFC	equ	$80	;Traffic flag
GMOV	equ	$40	;Game Over flag
PAD1	equ	$20	;Pad one flag
PAD2	equ	$10	;Pad two flag
CLSN1	equ	$08	;player 1 frog collision flag
CLSN2	equ	$04	;player 2 frog collision flag
PAS	equ	$02	;Pause flag
WIN	equ	$01	;Victory Flag

null	equ	$55	;Denotes the end of arrays (See Constants)

**LCD Addresses**
StreetPtr	equ	$001a	;Points to the beginning of the street
MidStreetPtr	equ	$0011	;Points to the beginning of the second street

Frog1		equ	$091c	;Starting location of player ones' frog
Frog2		equ	$061c	;Starting location of player two' frog

P1Life		equ	$0A1F	;Player one's life total location
P2Life		equ	$021F	;Player two's life total location


Row1		equ	$0200	;Default player one's frog location
Row2		equ	$0040	;Default player two's frog location


*=============
*= Variables =
*=============
	org	$0000

**Game state bits**
state	rmb	1	;Holds the state of the game
**bit7-Flag for periodic traffic movement, 1 => move cars one increment
**bit6-Flag for Player 1 pad, 1 => player 1 pad button pressed
**bit5-Flag for Player 2 pad, 1 => player 2 pad button pressed
**bit4-Flag for Game Over boolean, 1 => Game is Over
**bit3-Flag for player 1 frog collision, 1 => player 1 frog collided
**bit2-Flag for player 2 frog collision, 1 => player 2 frog collided
**bit1-Flag for Pause boolean, 1 => Pause the game
**bit0-Flag for Victory boolean, 1 => someone has won

frogState	rmb	1
**bit7-Flag for Player moved left frog
**bit6-Flag for Player moved down frog
**bit5-Flag for Player moved right frog
**bit4-Flag for Player moved up frog
**bit3-Flag for Dead player one, 1 => player one is dead
**bit2-Flag for Dead player two, 1 => player two is dead
**bit1-Flag for Player two WON!, 1 => player two won
**bit0-Flag for Player one WON!, 1 => player one won

**Used to create a period for the real-time interrupt**
trfcdly		rmb	1

**Controller Variables**
**Right buttons include:**
**Not Used**
*bit0-/\
*bit1-O
*bit2-X
*bit3-|_|
*bit4-L2
*bit5-R2
*bit6-L1
*bit7-R1
**Left buttons include:**
*bit0-Up
*bit1-Right
*bit2-Down
*bit3-Left
*bit4-Select
*bit5-n/a
*bit6-n/a
*bit7-Start

**Button State: high-byte=left buttons, low-byte=right buttons**
p1RightButtons	rmb	1
p1LeftButtons	rmb	1
p1ButtonState	rmb	2

p2RightButtons	rmb	1
p2LeftButtons	rmb	1
p2ButtonState	rmb	2

**LCD Variables**
cursor		rmb	2	;Address of the most recently set cursor

p1FrogPtr	rmb	2	;Pointer to player one's frog
p2FrogPtr	rmb	2	;Pointer to player two's frog

p1FrogRow	rmb	2	;Player one frog position data
p2FrogRow	rmb	2	;Player two frog position data

p1LastMv	rmb	2	;Holds the direction of player one's last move
p2LastMv	rmb	2	;Holds the direction of player two's last move

bottomLane	rmb	16	;Holds all traffic data for the bottom lane
topLane		rmb	16	;Holds all traffic data for the top lane

**Game Variables**
p1Lives		rmb	1	;Number of lives player one has
p2Lives		rmb	1	;Number of lives player two has


**Temporaries**
leftTemp	rmb	1
rightTemp	rmb	1

*========
*= Main =
*========
	org	$1000
Main:	
	sei	;Disable interrupts
	jsr	Init
	jsr	InitVar
	jsr	InitLCD

	jsr	DrawStage

	cli	;Enable interrupts

	jsr	Game

	brclr	state,GMOV,Game_Victory

Game_Over:
	sei

	
*Clear car blocks*
	ldd	#MidStreetPtr
	jsr	SetCursor
	ldy	#16
loop1:	jsr	Clear
	dbne	Y,loop1

	ldd	#MidStreetPtr+2
	jsr	SetCursor
	ldy	#16
loop2:	jsr	Clear
	dbne	Y,loop2


*Draw GAME OVER*
	ldx	#GAME_OVER
	ldd	#$0412
	jsr	DrawShape

	jmp	restart

Game_Victory:
	sei


*Clear car blocks*
	ldd	#MidStreetPtr+2
loop3:	jsr	SetCursor
	ldy	#16
loop4:	jsr	Clear
	dbne	Y,loop4
	ldd	cursor
	cpd	#MidStreetPtr-2
	beq	cont
	subd	#$01
	bra	loop3

cont:	brclr	frogState,#$01,plyr2


*Draw PLAYER 1 WINS!*
	ldx	#PLAYER1
	ldd	#$0410
	jsr	DrawShape
	ldx	#WINS
	ldd	#$0612
	jsr	DrawShape

	jmp	restart

*Draw PLAYER 2 WINS!*
plyr2:	ldx	#PLAYER2
	ldd	#$0410
	jsr	DrawShape
	ldx	#WINS
	ldd	#$0612
	jsr	DrawShape


*Restart the game once start has been pressed*
restart:	
	jsr	GetButtons
	jsr	CheckPads
	brclr	state,PAS,restart
	
	jmp	Main
	

*==========================
*= Initialize the program =
*==========================
Init:		pshb
		movb	#$ff,ddrh	;Set all bits in porth to output
		movb	#$ff,ddrg	;Set all bits in portg to output
		movb	#$00,porth	;Set all data bits inactive
		movb	#$1f,portg	;Set all commands inactive

		movb	#$87,rtictl	;Enable real-time interrupts and set the period to 63 ms
		movb	#$80,rtiflg	;clear and request a real-time interrupt upon period time-out

		movb	#$E0,ddrs	;set /SS, sclk, and MOSI to ouput and MISO to input
		movb	#$01,ddrt	;Set /SS to output
		movb	#$5D,spocr1	;set up SPI stuff
		movb	#$06,spobr	;set spi baud rate 64kHz
		bset	ports,#$80	;Disable Pad 1
		bset	portt,#$01	;Disable Pad 2

doneInit:	pulb
		rts

*Initialize variables
InitVar:	pshd
		pshx
		pshy

		clr	state		;Clear the game state byte
		clr	frogState	;Clear the frog byte
		clr	trfcdly		;clear the second delay

*Clear all buttons data*
		clr	p1LeftButtons
		clr	p1RightButtons
		clr	p1ButtonState
		clr	p2LeftButtons
		clr	p2RightButtons
		clr	p2ButtonState

*Load topLane with the initial state of the top road*
*These are manually loaded, because I was tired, and didn't feel like thinking*
		ldx	#bottomLane
		
		ldd	#$c003
		std	0,x
		ldd	#$d8f3
		std	2,x
		ldd	#$18f0
		std	4,x
		ldd	#$0300
		std	6,x
		ldd	#$e00e
		std	8,x
		ldd	#$e000
		std	10,x
		ldd	#$0f3e
		std	12,x
		ldd	#$0f3e
		std	14,x

		ldx	#topLane

		ldd	#$00e0
		std	0,x
		ldd	#$1ce0
		std	2,x
		ldd	#$dc06
		std	4,x
		ldd	#$0186
		std	6,x
		ldd	#$0180
		std	8,x
		ldd	#$7838
		std	10,x
		ldd	#$7b3b
		std	12,x
		ldd	#$0303
		std	14,x
		

*Set All frog data to default starting positions*
		ldab	#8
		stab	p1Lives
		stab	p2Lives
	
		ldd	#Frog1
		std	p1FrogPtr
		ldd	#Row1
		std	p1FrogRow
	
		ldd	#Frog2
		std	p2FrogPtr
		ldd	#Row2
		std	p2FrogRow
	
		ldd	#FROG_UP
		Std	p1LastMv
		std	p2LastMv

doneInitVar:	puly
		pulx
		puld
		rts


*==============
*= Game Logic =
*==============

*Controls all aspects of the Frogger Game
Game:		pshd
		pshx
		pshy


**If the game is over, end the game subroutine**
gameLoop:	tst	p1Lives
		bne	notOver
		tst	p2Lives
		bne	notOver

		bset	state,GMOV	;Set Game Over bit
		jmp	doneGame	;Game is over jump out!	


**Otherwise continue by grabbing the players buttons**
notOver:	jsr	GetButtons
		jsr	CheckPads
	
		brclr	state,PAS,chkTRFC	;Check Pause
		bclr	state,PAS
		sei
		jsr	PauseGame	;Draw the pause screen and wait for the start button
		cli


*Check for traffic movement*
chkTRFC:	brclr	state,TRFC,chkP1
		bclr	state,TRFC	;clear the real-time interrupt bit
		jsr	Traffic
		jsr	ClearBorder
		

**Check player one's status**
chkP1:		brset	frogState,#$08,chkP2	;Check if player one is dead
	
		brclr	state,PAD1,chkP2	;Check if player one pressed a button
		ldx	#p1FrogPtr
		ldy	#p1FrogRow
		ldd	p1ButtonState
	
		bclr	frogState,#$F0		;Clear the previous frogger direction
		jsr	MoveFrog
		bclr	state,PAD1
	
		jsr	RedrawStage		;Keeps frogger from deleting highway lines


*Check which direction frogger went, and rotate accordingly*
		brclr	frogState,#$80,p1Dwn
		ldd	#FROG_LEFT
		std	p1LastMv
	
p1Dwn:		brclr	frogState,#$40,p1Rht
		ldd	#FROG_DOWN
		std	p1LastMv
	
p1Rht:		brclr	frogState,#$20,p1Up
		ldd	#FROG_RIGHT
		std	p1LastMv
	
p1Up:		brclr	frogState,#$10,p1Win
		ldd	#FROG_UP
		std	p1LastMv

*Check if Player one has won*
p1Win:		brclr	state,WIN,chkP2
		bset	frogState,#$01	;Set player one's victory!
		jmp	doneGame	;End the game


**Check player two's status**
chkP2:		brset	frogState,#$04,redraw	;Check if player two is dead
	
		brclr	state,PAD2,redraw	;Check if player two pressed a button
		ldx	#p2FrogPtr
		ldy	#p2FrogRow
		ldd	p2ButtonState
	
		bclr	frogState,#$F0		;Clear the previous frogger direction
		jsr	MoveFrog
		bclr	state,PAD2
	
		jsr	RedrawStage		;Keeps frogger from deleting highway lines


*Check which direction frogger went, and rotate accordingly*
		brclr	frogState,#$80,p2Dwn
		ldd	#FROG_LEFT
		std	p2LastMv

p2Dwn:		brclr	frogState,#$40,p2Rht
		ldd	#FROG_DOWN
		std	p2LastMv
	
p2Rht:		brclr	frogState,#$20,p2Up
		ldd	#FROG_RIGHT
		std	p2LastMv
	
p2Up:		brclr	frogState,#$10,p2Win
		ldd	#FROG_UP
		std	p2LastMv

*Check if Player two has won*
p2Win:		brclr	state,WIN,clsnCk
		bset	frogState,#$02	;Set player two's victory!
		jmp	doneGame	;End the game


**Redraw frogs after every cycle, even if no button was pressed**

*Prevents Traffic subroutine from erasing a frog*
redraw:		ldd	p1FrogPtr
		ldx	p1LastMv
		jsr	DrawShape
		ldd	p2FrogPtr
		ldx	p2LastMv
		jsr	DrawShape


**Check for a collision**
clsnCk:		jsr	CollisionChk


*Check player one*
		brclr	state,CLSN1,cls2
		bclr	state,CLSN1


*Redraw life gauge for player one*
		dec	p1Lives		;Take a life away
		ldx	#NUMBERS
		ldab	p1Lives

*Multiply player one's score by 9 to get the number address from NUMBERS*
		tba
		lslb
		lslb
		lslb
		aba
		tab
		abx	
	
		ldd	#P1Life
		jsr	DrawShape
		
		tst	p1Lives		;Check if player one is dead
		beq	p1Dead
	
*Reset Player one's Frog*
		ldd	#Frog1
		std	p1FrogPtr
		ldd	#Row1
		std	p1FrogRow
		ldd	#FROG_UP
		std	p1LastMv
	
		jmp	cls2

*If player one is dead put him at the bottom of the screen*
p1Dead:		bset	frogState,#$08	;Set Player one to dead

		ldd	#$0B1F
		std	p1FrogPtr
		ldd	#FROG_UP
		std	p1LastMv
	

*Check player two*
cls2:		brclr	state,CLSN2,repeat
		bclr	state,CLSN2
	
		dec	p2Lives		;Take a life away
		ldx	#NUMBERS
		ldab	p2Lives

*Multiply player one's score by 9 to get the number address from NUMBERS*
		tba
		lslb
		lslb
		lslb
		aba
		tab
		abx
	
		ldd	#P2Life
		jsr	DrawShape

		tst	p2Lives		;Check if player two is dead
		beq	p2Dead
	
*Reset Player two's Frog*
		ldd	#Frog2
		std	p2FrogPtr
		ldd	#Row2
		std	p2FrogRow
		ldd	#FROG_UP
		std	p2LastMv
	
		jmp	repeat

*If player two is dead put him at the bottom of the screen*
p2Dead:		bset	frogState,#$04	;Set player two dead flag

		ldd	#$031F
		std	p2FrogPtr
		ldd	#FROG_UP
		std	p2LastMv

repeat:		jmp	gameLoop

doneGame:	puly
		pulx
		puld
		rts


*Checks for a collisions between vehicles and frogs
CollisionChk:	pshd
		pshx


*Find player ones frog*

**Lower byte of frogptr is the Y coordinate on the LCD**
		ldd	p1FrogPtr	;Load Register D with p1FrogPtr
		clra


**Check If player one is above StreetPtr and below MidStreetPtr**
		cpd	#StreetPtr	
		bgt	clsnP2
		cpd	#StreetPtr-7
		blt	midStreet1


**Is so, they are in the first street**

**Subtract p1FrogPtr from StreetPtr and two's complement to find the corresponding Row in bottomLane**

*(Two's complement is necessary because StreetPtr is larger than p1FrogPtr)*
		subd	#StreetPtr
		comb
		addb	#1



**Multiply the result by two for 2 byte increments**

**AND player one's row to the corresponding row in bottomLane**

**If any 1's occur, a collision has occurred**
		lslb
		ldx	#bottomLane
		abx
		ldd	p1FrogRow
		anda	0,x
		bne	colsn1
		andb	1,x
		bne	colsn1

		bra	clsnP2

colsn1:		bset	state,CLSN1

		bra	clsnP2


*Repeat for second street*
midStreet1:	ldd	p1FrogPtr
		clra
		cpd	#MidStreetPtr
		bgt	clsnP2
		cpd	#MidStreetPtr-7
		blt	clsnP2

		subd	#MidStreetPtr
		comb
		addb	#1
		lslb

		ldx	#topLane
		abx
		ldd	p1FrogRow
		anda	0,x
		bne	colsn1
		andb	1,x
		bne	colsn1


*Repeat for Player two*
clsnP2:		ldd	p2FrogPtr
		clra
		cpd	#StreetPtr
		bgt	doneColsnChk
		cpd	#StreetPtr-7
		blt	midStreet2

		subd	#StreetPtr
		comb
		addb	#1
		lslb

		ldx	#bottomLane
		abx
		ldd	p2FrogRow
		anda	0,x
		bne	colsn2
		andb	1,x
		bne	colsn2

		bra	doneColsnChk

colsn2:		bset	state,CLSN2

		bra	doneColsnChk

midStreet2:	ldd	p2FrogPtr
		clra
		cpd	#MidStreetPtr
		bgt	doneColsnChk
		cpd	#MidStreetPtr-7
		blt	doneColsnChk

		subd	#MidStreetPtr
		comb
		addb	#1
		lslb

		ldx	#topLane
		abx
		ldd	p2FrogRow
		anda	0,x
		bne	colsn2
		andb	1,x
		bne	colsn2	

doneColsnChk:	pulx
		puld
		rts


*Pauses the game and waits for the start button
PauseGame:	pshd
		pshx


*Draw the pause screen*
		ldx	#PAUSE
		ldd	#$0612
		jsr	DrawShape


*Check for the start button*
pasLoop1:	jsr	GetButtons
		jsr	CheckPads

		brclr	state,PAS,pasLoop1
		bclr	state,PAS


*Clear pause screen*
		ldd	#MidStreetPtr+1
		jsr	SetCursor
		ldy	#16
pasLoop2:	jsr	Clear
		dbne	Y,pasLoop2

donePause:	pulx
		puld
		rts	


*===================
*= LCD Subroutines =
*===================

*Moves traffic in the game
Traffic:	pshd
		pshx
		pshy

***Bottom Lane***
**Shift all cars one space to the right**

*Grab the state of the current row*

*Shift the entire row to move the traffic one space*
bottom:		ldx	#bottomLane
traffic1:	ldd	0,x
		lsrd



*If the carry bit is set (or a piece of a vehicle has moved off screen)*

*AND Reg A with 1000 to wrap the car back around*
		bcc	traffic2
		oraa	#$80



*Store the shifted bits back and Increment X by two bytes to the next row*
traffic2:	std	2,x+
		cpx	#bottomLane+16	
		bne	traffic1	

**Redraw shifted cars**
		ldd	#StreetPtr	
		ldx	#bottomLane



*Set cursor to the StreetPtr, and send the write command*
traffic3:	jsr	SetCursor
		ldab	#$42
		jsr	LCDCommand



*Draws cars by taking the state of each lane*

*Shifts each bit into the carry bit*

*If C=1 draw a square block at that memory location*
		ldy	#16
		ldd	2,x+
traffic4:	lsrd			;Shift the next bit into carry
		bcc	traffic5
		jsr	DrawSquare	;If the carry bit is set, draw a square at the LCD address in memory
		bra	traffic6
traffic5:	jsr	Clear		;Else Clear the LCD address in memory
traffic6:	dbne	Y,traffic4	;Check if all 16 bits of the whole row have been drawn	
		ldd	cursor		;Load the current cursor location
		cpd	#StreetPtr-7	;And compare it to the end of the top lane of the street
		beq	top	
		subd	#$01		;If not done move the cursor up one LCD block
		bra	traffic3

**Top Lane**
*Shift all cars one space to the left*
top:		ldx	#topLane
trfc1:		ldd	0,x
		lsld
		bcc	trfc2
		orab	#$01
trfc2:		std	2,x+
		cpx	#topLane+16
		bne	trfc1

*Redraw shifted cars*
		ldd	#MidStreetPtr
		ldx	#topLane
trfc3:		jsr	SetCursor
		ldab	#$42
		jsr	LCDCommand
		ldy	#16
		ldd	2,x+
trfc4:		lsrd
		bcc	trfc5
		jsr	DrawSquare
		bra	trfc6
trfc5:		jsr	Clear
trfc6:		dbne	Y,trfc4
		ldd	cursor
		cpd	#MidStreetPtr-7
		beq	doneTraffic
		subd	#$01
		bra	trfc3

doneTraffic:	puly
		pulx
		puld
		rts


*Clears the borders of the screen

*Gives a cleaner feel to the cars coming off and on screen*

*Also done because of the odd scrolling that occurs*
ClearBorder:	pshd

		ldd	#StreetPtr+1
ClrBrdrLoop1:	jsr	SetCursor
		ldy	#2
ClrBrdrLoop2:	jsr	Clear
		dbne	Y,ClrBrdrLoop2
		ldd	cursor
		cpd	#StreetPtr-18
		beq	doneClrBorder
		subd	#$01
		bra	ClrBrdrLoop1
		
doneClrBorder:	puld
		rts


*Moves a given frog based on controller input
**Input: Register X with the address of the Frog pointer
**Input: Register Y with the address of the Frog row
**Input: Register D with the LCD address
MoveFrog:	pshd
		pshx
		pshy


*Store left and right buttons into temporary variables*
		staa	leftTemp
		stab	rightTemp
		ldd	0,x		;Set cursor to the given frog pointer
		jsr	SetCursor


**Check Left button**
chkLeft:	brclr	leftTemp,#$80,chkDown
		inca			;Move frog pointer to the left one
		cmpa	#$0e		;Check left border
		lbeq	doneMoveFrog
		
**Check other frog position**
		cpd	p1FrogPtr	;Check for player one's frog
		lbeq	doneMoveFrog

		cpd	p2FrogPtr	;Check for player two's frog
		lbeq	doneMoveFrog

**Valid Move**
		std	0,x		;Store the new Frog pointer
		ldd	0,y		;Set the new frog row
		lsld
		std	0,y		;Store the new frog row
		
		ldd	0,x		;Send the frog pointer to DrawFrog
		jsr	Clear
		ldx	#FROG_LEFT	;Rotate the frog left
		jsr	DrawShape

		bset	frogState,#$80	;Set frog direction

		jmp	doneMoveFrog


**Check down button**
chkDown:	brclr	leftTemp,#$40,chkRight
		incb			;Move frog down one
		cmpb	#$1d		;Check bottom border
		lbeq	doneMoveFrog

**Check other frog position**
		cpd	p1FrogPtr	;Check for player one's frog
		lbeq	doneMoveFrog

		cpd	p2FrogPtr	;Check for player two's frog
		lbeq	doneMoveFrog

**Valid Move**
		std	0,x
		jsr	Clear
		ldx	#FROG_DOWN	;Rotate the frog down
		jsr	DrawShape

		bset	frogState,#$40	;Set frog direction

		jmp	doneMoveFrog


**Check Right button**
chkRight:	brclr	leftTemp,#$20,chkUp
		deca			;Move frog right one
		cmpa	#$01		;Check right border
		beq	doneMoveFrog

**Check other frog position**
		cpd	p1FrogPtr	;Check for player one's frog
		beq	doneMoveFrog

		cpd	p2FrogPtr	;Check for player two's frog
		beq	doneMoveFrog

**Valid Move**
		std	0,x		;Store the new Frog pointer
		ldd	0,y		;Set the new frog row
		lsrd
		std	0,y		;Store the new frog row
		
		ldd	0,x		;Send the frog pointer to DrawFrog
		jsr	Clear
		ldx	#FROG_RIGHT	;Rotate the frog right
		jsr	DrawShape
	
		bset	frogState,#$20	;Set Frog direction

		jmp	doneMoveFrog

**Check up button**
chkUp:		brclr	leftTemp,#$10,doneMoveFrog
		decb			;Move frog up one
		cmpb	#$07		;Check finish line
		bne	upChk		;If not at the finish line continue as usual

		bset	state,WIN

**Check other frog position**
upChk:		cpd	p1FrogPtr	;Check for player one's frog
		beq	doneMoveFrog

		cpd	p2FrogPtr	;Check for player two's frog
		beq	doneMoveFrog

**Valid Move**
		std	0,x
		jsr	Clear
		ldx	#FROG_UP	;Rotate the frog up
		jsr	DrawShape

		bset	frogState,#$10	;Set frog direction

doneMoveFrog:	puly
		pulx
		puld
		rts


*Draws a Shape at the given LCD address
**Input: Register X with pointer to the shape constant
**Input: Register D with starting LCD address
DrawShape:	pshx
		pshd

		jsr	SetCursor	
		ldab	#$42
		jsr	LCDCommand
shapeLoop:	ldab	1,x+
		cmpb	#null
		beq	doneDrawShp
		jsr	LCDData
		bra	shapeLoop


doneDrawShp:	puld
		pulx
		rts
	

*Used to draw a shape on an entire row in memory
**Input: Register X with the pointer to the shape constant
DrawRow:	pshx
		pshd
	
		ldab	#$42
		jsr	LCDCommand
rowLoop:	ldab	1,x+
		cmpb	#null
		beq	doneDrawRow
		jsr	LCDData
		bra	rowLoop	

doneDrawRow:	puld
		pulx
		rts	


*Draws a square at the LCD address in memory
DrawSquare:	pshd
		pshx
		ldab	#$42
		jsr	LCDCommand
		ldaa	#$8
sqrLoop:	ldab	#$ff
		jsr	LCDData
		deca
		bne	sqrLoop
doneDrawSqr:	pulx
		puld
		rts


*Clears the screen at the LCD address in memory
Clear:		pshd
		pshx
		ldab	#$42
		jsr	LCDCommand
		ldaa	#$8
clrLoop:	ldab	#$00
		jsr	LCDData
		deca
		bne	clrLoop
doneClear:	pulx
		puld
		rts


*Sets the Cursor address
**Input: register D with the LCD address
**Return: cursor, with the most recently set cursor position
SetCursor:	pshd
		pshb
		ldab	#$46
		jsr	LCDCommand
		pulb
		std	cursor	;store the cursor
		jsr	LCDData	;Send lower byte
		tab	
		jsr	LCDData	;Send higher byte
doneSetCursor:	puld
		rts


*=========================
*= PSOne Pad Subroutines =
*=========================

*Grab the states of each players controllers
**Grabs the state of the player one's controller**
GetButtons:	pshd

		bclr	ports,#$80	;Enable P1 Pad (negative logic)
		
		ldab	#$01		;Say Hello to the Pad
		jsr	PadCommand
		ldab	#$42		;Request Pad ID
		jsr	PadCommand	;Pad sent back, $41 for standard digital pad

*No more commands needed*		
		jsr	PadCommand	;Here comes the data, $5A will be sent
		jsr	PadCommand	;D-Pad, start, and select have been sent
		comb			;Given in negative logic, convert it to positive
		bne	p1DPadStrtSlc
		clr	p1LeftButtons
		bra	p1GetNxtBtns
p1DPadStrtSlc:	stab	p1LeftButtons	;Store D-Pad, start, and select data in leftButtons

p1GetNxtBtns:	jsr	PadCommand	;Shapes and triggers
		comb
		bne	p1ShapesLR
		clr	p1RightButtons
		bra	GetP2Buttons
p1ShapesLR:	stab	p1RightButtons	;Store Shapes and Triggers data in rightButtons


**Grabs the state of the player two's controller**
GetP2Buttons:	bset	ports,#$80	;Disable P1 Pad

		bclr	portt,#$01	;Enable P2 Pad (negative logic)
		
		ldab	#$01		;Say Hello to the Pad
		jsr	PadCommand
		ldab	#$42		;Request Pad ID
		jsr	PadCommand	;Pad sent back, $41 for standard digital pad

*No more commands needed*		
		jsr	PadCommand	;Here comes the data, $5A will be sent
		jsr	PadCommand	;D-Pad, start, and select have been sent
		comb			;Given in negative logic, convert it to positive
		bne	p2DPadStrtSlc
		clr	p2LeftButtons
		bra	p2GetNxtBtns
p2DPadStrtSlc:	stab	p2LeftButtons	;Store D-Pad, start, and select data in leftButtons

p2GetNxtBtns:	jsr	PadCommand	;Shapes and triggers
		comb
		bne	p2ShapesLR
		clr	p2RightButtons
		bra	doneGetBtns
p2ShapesLR:	stab	p2RightButtons	;Store Shapes and triggers in rightButtons

doneGetBtns:	bset	portt,#$01	;Disable P2 Pad
		puld
		rts
		

*Sends the Pad a command
**Input: Register B with the command
**Return: Register B with Pad data
PadCommand:	psha
		stab	spodr
padCmdLoop:	ldab	sposr
		andb	#$80
		beq	padCmdLoop
		ldab	spodr
donePadCmd:	pula
		rts


*Checks the controllers for user input
CheckPads:	pshd

**Check for new data from player one's controller**
		ldd	p1ButtonState	;Load the previous state of the controller



*Compare previous state to current state*
		cmpb	p1RightButtons
		bne	p1NewState

		cmpa	p1LeftButtons
		bne	p1NewState

		jmp	checkP2Pad	;jump if nothing has changed

**Store the new state of player one's controller**
p1NewState:	bset	state,PAD1	;P1 Pressed a button

		ldaa	p1LeftButtons
		ldab	p1RightButtons
		std	p1ButtonState

**Check Pause button**
		brclr	p1LeftButtons,#$08,checkP2Pad
		bset	state,PAS

**Check for new data from player two's controller**
checkP2Pad:	ldd	p2ButtonState	;Load previous state of the controller



*Compare previous state  to current state*
		cmpb	p2RightButtons
		bne	p2NewState

		cmpa	p2LeftButtons
		bne	p2NewState
		
		jmp	doneCheckPads	;Jump if nothing has changed

**Store the new state of player two's controller**
p2NewState:	bset	state,PAD2	;P2 pressed a button

		ldaa	p2LeftButtons
		ldab	p2RightButtons
		std	p2ButtonState

**Check Pause button**
		brclr	p2LeftButtons,#$08,doneCheckPads
		bset	state,PAS

doneCheckPads:	puld
		rts


*======================
*= Initialize the LCD =
*======================
InitLCD:	pshb
		pshx

		bclr	porth,#$01	;/Reset=0

***Need at least 3ms delay***
		ldx	#$ffff
lcdLoop1:	dex
		bne	lcdLoop1

		bset	porth,#$01	;/Reset=1, Reset complete 

***Need at least 50 ms delay***
		ldx	#$ffff
lcdLoop2:	dex
		bne	lcdLoop2

		ldab	#$58	;Turn display off
		jsr	LCDCommand
		
**Initial Setup**
		ldab	#$40	;Initialize device and display
		jsr	LCDCommand
		ldab	#$30	;Internal CG ROM, CG RAM1; 32 char, 8-pixel char height, single-panel drive, IV=1, LCD Mode
		jsr	LCDData
		ldab	#$87	;Two frame AC drive and 8-pixel char width FX
		jsr	LCDData
		ldab	#$07	;8-pixel Char height FY
		jsr	LCDData
		ldab	#$1f	;32 display bytes per line
		jsr	LCDData
		ldab	#$23	;Total address range per line TC/R (C/R+4 H-Blanking)
		jsr	LCDData
		ldab	#$74	;128 display lines L/F+1
		jsr	LCDData
		ldab	#$20	;Sets horz addr range APLow (Virtual Screen)
		jsr	LCDData
		ldab	#$00	;APHigh (Virtual Screen)
		jsr	LCDData

**Scroll Setup**
		ldab	#$44	;Set Display start address and regions
		jsr	LCDCommand
		ldab	#$00	;start address 1 lower byte
		jsr	LCDData
		ldab	#$00	;start address 1 high byte
		jsr	LCDData
		ldab	#$7f	;128 lines per scrolling screen 1
		jsr	LCDData
		ldab	#$00	;start address 2 lower byte
		jsr	LCDData
		ldab	#$10	;start address 2 high byte
		jsr	LCDData
		ldab	#$7f	;128 lines per scrolling screen 2
		jsr	LCDData
		ldab	#$00	;start address 3 lower byte
		jsr	LCDData
		ldab	#$20	;start address 3 high byte
		jsr	LCDData
		ldab	#$7f	;128 lines per scrolling screen 3
		jsr	LCDData

**Horizontal Scroll Setup**
		ldab	#$5a	;Set Horizontal scroll position
		jsr	LCDCommand
		ldab	#$00	;At Origin on X
		jsr	LCDData

**Overlay Setup**
		ldab	#$5b	;Set Display Overlay Format screen text/graphics mode
		jsr	LCDCommand
		ldab	#$1c	;Layered screen composition method=OR, Graphics mode, and 3-layered graphics mode
		jsr	LCDData

**Cursor setup**
		ldab	#$4c	;Automatic cursor increment down
		jsr	LCDCommand
		
		ldab	#$46	;Set cursor address location
		jsr	LCDCommand
		ldab	#$00	;lower cursor byte
		jsr	LCDData
		ldab	#$00	;high curosr byte
		jsr	LCDData

**Clear Memory**
		ldx	#$0000
		ldab	#$42	;Write to display memory
		jsr	LCDCommand
clrMemLp:	clrb
		jsr	LCDData
		inx
		cpx	#$3000
		bne	clrMemLp

*Set Cursor increment again*
		ldab	#$4f
		jsr	LCDCommand
		
**Turn on display**
		ldab	#$59	;Turn Display on
		jsr	LCDCommand
		ldab	#$14	;No layer 3 or 4, Show layer 1 and 2, No cursor
		jsr	LCDData
	
doneInitLCD:	pulx
		pulb
		rts




**LCD Bits**
*/Reset	equ	$01
*/Read	equ	$02
*/Write	equ	$04
*/CS	equ	$08
*A0	equ	$10

*Send command to the LCD
**Input: Register B
LCDCommand:	pshb
		bset	porth,#$10	;Set A0=1
		stab	portg		;Send Command
		bset	porth,#$02	;Disable /Read
		bclr	porth,#$08	;Enable /CS
		bclr	porth,#$04	;Enable /Write
		movb	#$0f,porth	;Disable /Write and /CS
doneCmd:	pulb
		rts


*Send data to the LCD
**Input: Register B
LCDData:	pshb
		bclr	porth,#$10	;Set A0=0
		stab	portg		;Send Data
		bset	porth,#$02	;Disable /Read
		bclr	porth,#$08	;Enable /CS
		bclr	porth,#$04	;Enable /Write
		movb	#$0f,porth	;Disable /Write and /CS
doneData:	pulb
		rts


*Draw the stage
DrawStage:	pshd
		pshx


**Draws the border around FROGGER**
		ldx	#TOP_BORDER
		ldd	#$0403
stgLoop1:	jsr	DrawShape
		inca
		cmpa	#$0d
		bne	stgLoop1

		ldx	#BOTTOM_BORDER
		ldd	#$0407
stgLoop2:	jsr	DrawShape
		inca
		cmpa	#$0d
		bne	stgLoop2

		ldx	#RIGHT_BORDER
		ldd	#$0406
stgLoop3:	jsr	DrawShape
		decb
		cmpb	#$03
		bne	stgLoop3

		ldx	#LEFT_BORDER
		ldd	#$0c06
stgLoop4:	jsr	DrawShape
		decb
		cmpb	#$03
		bne	stgLoop4


*Draw FROGGER*
		ldx	#FROGGER
		ldd	#$0505
		jsr	DrawShape


*Draw P1 life*
		ldx	#P1
		ldd	#$0C1F
		jsr	DrawShape


*Draw P2 life*
		ldx	#P2
		ldd	#$041F
		jsr	DrawShape

		ldx	#NUMBERS
		ldab	p1Lives
		tba
		lslb
		lslb
		lslb
		aba
		tab
		abx

		ldd	#P2Life
		jsr	DrawShape
		ldd	#P1Life
		jsr	DrawShape


*Draw Signature*
		ldx	#Zach
		ldd	#$0500
		jsr	DrawShape
		ldx	#Chris
		ldd	#$0200
		jsr	DrawShape


*Draw erasable shapes*
		jsr	RedrawStage
	
doneDrawStg:	pulx
		puld
		rts
	

*Redraws erasable parts of the screen
RedrawStage:	pshd
		pshx
	
		ldx	#ROAD_HASH
		ldd	#MidStreetPtr+1
		jsr	SetCursor
		ldy	#16
reLoop1:	jsr	DrawRow
		dbne	Y,reLoop1

		ldx	#TOP_BORDER
		ldd	#MidStreetPtr-8
		jsr	SetCursor
		ldy	#16
reLoop2:	jsr	DrawRow
		dbne	Y,reLoop2

		ldx	#BOTTOM_BORDER
		ldd	#StreetPtr+1
		jsr	SetCursor
		ldy	#16
reLoop3:	jsr	DrawRow
		dbne	Y,reLoop3

		jsr	ClearBorder	

doneRedrawStg:	pulx
		puld
		rts


*==============
*= Interrupts =
*==============
	
*Interrupt triggered every 63 ms
isr_rti:	bset	rtiflg,#$80
		
		inc	trfcdly
		ldab	trfcdly


*Only move traffic every 378 ms*
		cmpb	#7 
		bne	done_rti
		bset	state,TRFC

		clr	trfcdly
done_rti:	rti


*=============
*= Constants =
*=============
	org	$2000

**Road graphics**
ROAD_HASH:
	fcb	$00,$18,$18,$18,$18,$18,$18,$00	;Road hash shape
	fcb	null

TOP_BORDER:
	fcb	$01,$01,$01,$01,$01,$01,$01,$01	;Line on top
	fcb	null

BOTTOM_BORDER:
	fcb	$80,$80,$80,$80,$80,$80,$80,$80	;Line on bottom
	fcb	null

LEFT_BORDER:
	fcb	$00,$00,$00,$00,$00,$00,$00,$FF	;Line on left
	fcb	null

RIGHT_BORDER:
	fcb	$FF,$00,$00,$00,$00,$00,$00,$00	;Line on right
	fcb	null


****Loaded Manually****
BOTTOM_LANE:
	fcb	$0F,$3E,$0F,$3E,$E0,$00,$E0,$0E,$03,$00,$18,$F0,$D8,$F3,$C0,$03
	fcb	null


****Loaded Manually****
TOP_LANE:
	fcb	$03,$03,$7B,$3B,$78,$38,$01,$80,$01,$86,$DC,$06,$1C,$E0,$00,$E0
	fcb	null


**Frog Sprites**
FROG_UP:
	fcb	$42,$EF,$38,$7C,$7C,$38,$EF,$42	
	fcb	null

FROG_DOWN:
	fcb	$42,$F7,$1C,$3E,$3E,$1C,$F7,$42
	fcb	null

FROG_LEFT:
	fcb	$42,$C3,$5A,$7E,$3C,$7E,$DB,$42
	fcb	null

FROG_RIGHT:
	fcb	$42,$DB,$7E,$3C,$7E,$5A,$C3,$42
	fcb	null



**Words**
SPACE:
	fcb	$00,$00,$00,$00,$00,$00,$00,$00	;[SPACE]
	fcb	null

FROGGER:
	fcb	$00,$00,$71,$8A,$8C,$88,$88,$FF	;R
	fcb	$00,$00,$81,$81,$91,$91,$91,$FF	;E
	fcb	$00,$00,$6E,$89,$89,$81,$81,$7E	;G
	fcb	$00,$00,$6E,$89,$89,$81,$81,$7E	;G
	fcb	$00,$00,$7E,$81,$81,$81,$81,$7E	;O
	fcb	$00,$00,$71,$8A,$8C,$88,$88,$FF	;R
	fcb	$00,$00,$80,$80,$90,$90,$90,$FF	;F
	fcb	null

GAME_OVER:
	fcb	$00,$00,$71,$8A,$8C,$88,$88,$FF	;R
	fcb	$00,$00,$81,$81,$91,$91,$91,$FF	;E
	fcb	$00,$00,$F8,$06,$01,$01,$06,$F8	;V
	fcb	$00,$00,$7E,$81,$81,$81,$81,$7E	;O
	fcb	$00,$00,$00,$00,$00,$00,$00,$00	;[SPACE]
	fcb	$00,$00,$81,$81,$91,$91,$91,$FF	;E
	fcb	$00,$00,$FF,$60,$10,$10,$60,$FF	;M
	fcb	$00,$00,$0F,$78,$88,$88,$78,$0F	;A
	fcb	$00,$00,$6E,$89,$89,$81,$81,$7E	;G	
	fcb	null

PAUSE:
	fcb	$00,$00,$12,$29,$29,$29,$29,$1E	;e
	fcb	$00,$00,$12,$25,$29,$29,$29,$12	;s
	fcb	$00,$00,$3E,$01,$01,$01,$01,$3E	;u
	fcb	$00,$00,$01,$1e,$29,$29,$29,$16	;a
	fcb	$00,$00,$70,$88,$88,$88,$88,$7F	;P
	fcb	null

P1:
	fcb	$00,$00,$01,$01,$FF,$41,$21,$01	;1
	fcb	$00,$00,$70,$88,$88,$88,$88,$7F	;P
	fcb	null

P2:
	fcb	$00,$00,$61,$91,$89,$85,$83,$41	;2
	fcb	$00,$00,$70,$88,$88,$88,$88,$7F	;P
	fcb	null

PLAYER1:
	fcb	$00,$00,$01,$01,$FF,$41,$21,$01	;1
	fcb	$00,$00,$00,$00,$00,$00,$00,$00	;[SPACE]
	fcb	$00,$00,$71,$8A,$8C,$88,$88,$FF	;R
	fcb	$00,$00,$81,$81,$91,$91,$91,$FF	;E
	fcb	$00,$80,$40,$20,$1F,$20,$40,$80	;Y
	fcb	$00,$00,$0F,$78,$88,$88,$78,$0F	;A
	fcb	$00,$00,$01,$01,$01,$01,$01,$FF	;L
	fcb	$00,$00,$70,$88,$88,$88,$88,$7F	;P
	fcb	null

PLAYER2:
	fcb	$00,$00,$61,$91,$89,$85,$83,$41	;2
	fcb	$00,$00,$00,$00,$00,$00,$00,$00	;[SPACE]
	fcb	$00,$00,$71,$8A,$8C,$88,$88,$FF	;R
	fcb	$00,$00,$81,$81,$91,$91,$91,$FF	;E
	fcb	$00,$80,$40,$20,$1F,$20,$40,$80	;Y
	fcb	$00,$00,$0F,$78,$88,$88,$78,$0F	;A
	fcb	$00,$00,$01,$01,$01,$01,$01,$FF	;L
	fcb	$00,$00,$70,$88,$88,$88,$88,$7F	;P
	fcb	null	

WINS:
	fcb	$00,$00,$00,$00,$00,$FD,$00,$00	;!
	fcb	$00,$00,$4E,$91,$91,$91,$91,$62	;S
	fcb	$00,$00,$FF,$06,$08,$10,$60,$FF	;N
	fcb	$00,$00,$00,$81,$81,$FF,$81,$81	;I
	fcb	$00,$00,$FF,$06,$08,$08,$06,$FF	;W
	fcb	null

NUMBERS:
	fcb	$00,$00,$7E,$A1,$91,$89,$85,$7E,null	;0
	fcb	$00,$00,$01,$01,$FF,$41,$21,$01,null	;1
	fcb	$00,$00,$61,$91,$89,$85,$83,$41,null	;2
	fcb	$00,$00,$6E,$91,$91,$91,$81,$42,null	;3
	fcb	$00,$00,$08,$FF,$08,$08,$08,$F8,null	;4
	fcb	$00,$00,$8E,$91,$91,$91,$91,$F2,null	;5
	fcb	$00,$00,$0E,$91,$91,$91,$51,$3E,null	;6
	fcb	$00,$00,$E0,$90,$8C,$83,$80,$C0,null	;7
	fcb	$00,$00,$6E,$91,$91,$91,$91,$6E,null	;8
	fcb	$00,$00,$7C,$8A,$89,$89,$89,$70,null	;9

Chris:
	fcb	$00,$00,$6E,$89,$89,$81,$81,$7E	;G.
	fcb	$00,$00,$66,$81,$81,$81,$81,$7E	;C.
	fcb	null

Zach:
	fcb	$00,$00,$FF,$06,$08,$08,$06,$FF	;W.
	fcb	$00,$00,$c1,$a1,$91,$89,$85,$83	;Z.
	fcb	null



*=====================
*= Interrupt Vectors =
*=====================
*Real Time Interrupt*
	org	$630
	fdb	isr_rti