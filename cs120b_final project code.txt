/*
 * cs120B_final_project.c
 *
 * Created: 8/27/2018 6:55:47 PM
 * Author : Siwon Kim
 */ 
#include <avr/io.h>
#include <avr/interrupt.h>
#include "bit.h"
#include "timer.h"
#include <stdio.h>
#include <stdlib.h>
#include "io.h"
#include "io.c"
#include "scheduler.h"

#define start_button (~PIND&0x01)
#define right_button (~PIND&0x02)
#define left_button (~PIND&0x04)
#define right_button2 (~PIND&0x10)
#define left_button2 (~PIND&0x20)
#define reset_button (left_button && right_button)


unsigned char plyr_pos;     //playr index (start in middle)
unsigned char plyr_pos2;     //playr index (start in middle + 2)
unsigned char x_pos;
unsigned char x_pos_r1;
unsigned char x_pos_r2;
unsigned char y_pos;
unsigned char led_rowPos[] = {0x01,0x04,0x10,0x40};
unsigned char currObj;
unsigned char game_lost = 0;
unsigned char lives = 3;			//lives is 2 but concerning deplay 2+1 = 3;
unsigned char plyr2_arr[] = {0xFE, 0x7F};
unsigned char cntrVal = 50;
enum playr_states{init_plyr, move_playr};
int playr_char_sm(int state){
	static unsigned char cntr = 0; //cntr to slow down button
	switch(state){
		case init_plyr:
		plyr_pos = 0xEF;
		if ((start_button || !game_lost))
		{
			state = move_playr;
			
		}
		
		break;
		case move_playr:
		if(plyr_pos == 0x7F && right_button && cntr >= cntrVal){
			plyr_pos = 0xFE;
			cntr = 0;
		}
		else if(plyr_pos == 0xFE && left_button && !right_button && cntr >=cntrVal){
			plyr_pos = 0x7F;
			cntr = 0;
		}
		else if(right_button && !left_button && cntr >=cntrVal){
			plyr_pos = (plyr_pos << 1) | 1;
			cntr = 0; 
		}
		else if(!right_button && left_button && cntr >=cntrVal){
			plyr_pos = (plyr_pos >> 1) | 0x80;
			cntr = 0;
		}
		cntr++;
		if (game_lost)
		{
			state = init_plyr;
		}
		if (reset_button)
		{
			state = init_plyr;
			lives = 3;
		}
		break;
		default:	
		state = init_plyr;
		break;
	}
	return state;
}
enum playr_states2{init_plyr2, move_playr2};
int playr_char_sm2(int state){
	static unsigned char cntr = 0; //cntr to slow down button
	switch(state){
		case init_plyr2:
		plyr_pos2 = 0xFE;
		if ((start_button || !game_lost))
		{
			state = move_playr2;
			
		}
		
		break;
		case move_playr2:
		if(plyr_pos2 == 0x7F && right_button2 && cntr >=cntrVal/2){
			plyr_pos2 = 0xFE;
			cntr = 0;
		}
		else if(plyr_pos2 == 0xFE && left_button2 && !right_button2 && cntr >=cntrVal/2){
			plyr_pos2 = 0x7F;
			cntr = 0;
		}
		else if(right_button2 && !left_button2 && cntr >=cntrVal/2){
			plyr_pos2 = (plyr_pos2 << 1) | 1;
			cntr = 0;
		}
		else if(!right_button2 && left_button2 && cntr >=cntrVal/2){
			plyr_pos2 = (plyr_pos2 >> 1) | 0x80;
			cntr = 0;
		}
		cntr++;
		if (game_lost)
		{
			state = init_plyr2;
		}
		if (reset_button)
		{
			state = init_plyr2;
			lives = 3;
		}
		break;
		default:
		state = init_plyr2;
		break;
	}
	return state;
}

enum disp_plyrORobj{start_disp,disp_plyr,disp_plyr2, disp_obj};			//display between obj and the 
int alternate_display(int state){
	static unsigned char i = 0 ;
	switch(state){
		case start_disp:
// 		if (start_button && !game_lost)
// 		{
			state = disp_plyr;
//		}
		break;
		
		case disp_plyr:
		PORTA = 0x80; //bottom row
		PORTB = plyr_pos;	
		state = disp_plyr2; 
		break;

		case disp_plyr2:
		PORTA = 0x80; //bottom row
		PORTB = plyr_pos2;	
		state = disp_obj;
		break;
		
		case disp_obj:
		 PORTA = y_pos;
		 PORTB = x_pos;
		if (y_pos == 0x80) PORTB = x_pos & plyr_pos;			
		state = disp_plyr;	
			i++;
		break;
	}
	
	return state;
}
enum obj_states{obj_init, obj_Matdisp};
int obj_display_sm(int state){						//display the objects
static unsigned char selection_led;  //condition to output rows
	switch(state){
		case obj_init:
		selection_led = 0x01;

		state = obj_Matdisp;
		break;
		case obj_Matdisp:
// 		if (selection_led == 1)
// 		{	
// 			x_pos = x_pos_r1;
// 			y_pos = led_rowPos[0];
// 			currObj = 0;
// 
//  		}
// 		if (selection_led == 2)
// 		{
// 			x_pos = 0xFD;
// 			y_pos = led_rowPos[1];
// 			currObj = 1;

// 		}
		if (selection_led == 3)
		{
			y_pos = led_rowPos[2];
			x_pos =  x_pos_r2;
			currObj = 2;

		}
// 		if (selection_led == 4)
// 		{
// 			x_pos = 0xF7;
// 			y_pos = led_rowPos[3];
// 			currObj = 3;

// 		}

		if (selection_led >= 4) selection_led = 1;     //initialize row
		else{
			selection_led++;
		}
		break;
		if (game_lost || reset_button)
		{
			state = obj_init;
			selection_led = 1;
		}
	}
	return state;
}	
enum sftled_states{init_sft,sftdwn_led};
int shiftled_sm(int state){
	switch(state){
		case init_sft:
		if (start_button)
		{
			state = sftdwn_led;
		}
		break;
		
		case sftdwn_led:
		if (y_pos == 0x80 && !game_lost)
		{
			led_rowPos[currObj] = 0x01; //reset position
			x_pos_r1 = rand() % 8 + 20;
			x_pos_r2 = rand() % 8 + 30;
			plyr_pos2 = 0xFE;
		}
		else if(!game_lost){
			led_rowPos[currObj] = (led_rowPos[currObj] << 1);
		}
		if (reset_button)
		{
// 			led_rowPos[0] = 0x01;
// 			led_rowPos[1] = 0x04;
// 			led_rowPos[2] = 0x10;
// 			led_rowPos[3] = 0x40;
		}

		break;
	}
	return state;
}

enum game_states{check_result, gameLost_state};
int winLose_checkSM(int state){
	static unsigned char cntr;
	switch(state){
	case check_result:
	game_lost = 0;
	if (y_pos == 0x80)
	{
		if (((~x_pos | ~plyr_pos) == ~x_pos) && (lives > 0) && cntr >= 35)
		{
			lives--;
			cntr = 0;
			
		}
		if (((~x_pos | ~plyr_pos) == ~x_pos) && (lives > 0) && cntr >= 35)
		{
			lives--;
			cntr = 0;
			
		}
		if ((~x_pos | ~plyr_pos) == ~x_pos)
		{
			PORTD = PORTD ^ 0x08;
		}
// 		if (plyr_pos == plyr_pos2)
// 		{
// 			state = gameLost_state;
// 		}
		else if(lives == 0){
			state = gameLost_state;
		}
		cntr++;
	}
	break;
	
	case gameLost_state:
	game_lost = 1;					//set flag game lost
	if (reset_button)
	{
		lives = 3;
		state = check_result;
	}
	break;
	}
	return state;
	}

int main(void)
{
	DDRA = 0xFF; PORTA = 0x00;
	DDRB = 0xFF; PORTB = 0x00;
	DDRC = 0xFF; PORTC = 0x00;
	DDRD = 0x08; PORTD = 0xF7;
	
	unsigned long int plyr_TaskCalc = 2;
	unsigned long int disp_TaskCalc = 1;
	unsigned char sftPeriod = 240;
	unsigned long int tmpGCD = 1;
	tmpGCD = findGCD(plyr_TaskCalc, disp_TaskCalc);
	
	unsigned long int GCD = tmpGCD;
	
	unsigned long int plyr_Period = plyr_TaskCalc/GCD;
	unsigned long int disp_Period = disp_TaskCalc/GCD;
	
	static task plyrChar_task,plyrChar_task2, dispObj_task, plyrORobj_task, sftled_task, winlose_check_task;
	task *tasks[] = {&plyrChar_task,&plyrChar_task2, &sftled_task,&dispObj_task, &winlose_check_task, &plyrORobj_task};
	const unsigned short numTasks = sizeof(tasks)/sizeof(task*);
	
	plyrChar_task.state = init_plyr;
	plyrChar_task.period = plyr_Period*2;
	plyrChar_task.elapsedTime = plyr_Period*2;
	plyrChar_task.TickFct = &playr_char_sm;
	
	plyrChar_task2.state = init_plyr2;
	plyrChar_task2.period = plyr_Period*4;
	plyrChar_task2.elapsedTime = plyr_Period*4;
	plyrChar_task2.TickFct = &playr_char_sm2;
	
	plyrORobj_task.state = start_disp;
	plyrORobj_task.period = disp_Period;
	plyrORobj_task.elapsedTime = disp_Period;
	plyrORobj_task.TickFct = &alternate_display;
	
	dispObj_task.state = obj_init;
	dispObj_task.period = disp_Period;
	dispObj_task.elapsedTime = disp_Period;
	dispObj_task.TickFct = &obj_display_sm;

	sftled_task.state = init_sft;
	sftled_task.period = sftPeriod; //ms
	sftled_task.elapsedTime = sftPeriod; //ms
	sftled_task.TickFct = &shiftled_sm;
	
	winlose_check_task.state = check_result;
	winlose_check_task.period = disp_Period; //ms
	winlose_check_task.elapsedTime = disp_Period; //ms
	winlose_check_task.TickFct = &winLose_checkSM;
	LCD_init();
	LCD_ClearScreen();
	TimerSet(GCD);
	TimerOn();
	
	unsigned short i;
	
	while (1)
	{
		for(i = 0; i < numTasks; i++) {
			// task is ready to tick
			if(tasks[i]->elapsedTime == tasks[i]->period) {
				// setting next state for task
				tasks[i]->state = tasks[i]->TickFct(tasks[i]->state);
				// reset the elapsed time for the next tick
				tasks[i]->elapsedTime = 0;
			}
			tasks[i]->elapsedTime += 1;
		}

		while(!TimerFlag);      // wait for a period
		TimerFlag = 0;          // reset TimerFlag
		}
} 