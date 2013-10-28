/* CSCI 261 Lab/Homework 10: Yahtzee!
 *
 * Co-Author: Alex Patel
 *
 * Co-Author: _INSERT_YOUR_NAME_HERE_
 *
 * This program uses arrays to play the dice game Yahtzee.
 */

#include <iostream>
#include <iomanip>
#include <fstream>
#include <cstdlib>
#include <string>
#include <cmath>
#include <ctime>
#include "../lib/graphics.h"

using namespace std;

// All dimensions in this game are based on the size of a single die.
const int DIE_SIZE = 100;
const int MARGIN   = DIE_SIZE / 10;
const int WIDTH  = DIE_SIZE * 4 + MARGIN * 5;
const int HEIGHT = DIE_SIZE * 6 + MARGIN * 7;

// Function prototypes ... *** You Must Write These Functions *** ...
void sort( int dice[] );
int sumAll( const int dice[] );
int sumOnly( const int dice[], const int value );
bool isThreeOfAKind( const int dice[] );
bool isFourOfAKind( const int dice[] );
bool isFullHouse( const int dice[] );
bool isSmallStraight( const int dice[] );
bool isLargeStraight( const int dice[] );
bool isYahtzee( const int dice[] );
int getFinalScore( const int scores[] );

// Function prototypes ... These Functions Are Already Written ...
int getScore( int dice[], const int category );
void rollDice( int dice[], const bool hold[] );
void resetDice( int dice[], bool hold[] );
void drawDice( const int dice[], const bool hold[] );
void drawDie( const int value, const bool hold, const int x, const int y );
void drawScores( const int scores[] );
void drawButton( const int turns, const int rolls );
void waitForMouseClick( int &x, int &y );
int randomBetween( const int lowerBound, const int upperBound );

/*
 * The main program controls all of the game play.
 *
 *     ***********************************
 *     *** DO NOT MODIFY THIS FUNCTION ***
 *     ***********************************
 */
int main()
{
  // Seed the random number generator with the system clock.
  srand( (int)time( NULL ) );

  // This variable is used to control a BGI feature called "double-buffering".
  // *** You should not change any code that uses this variable! ***
  int page = 0;

  // Initialize a graphics window and set a few drawing properties.
  initwindow( WIDTH, HEIGHT, "Yahtzee", MARGIN, MARGIN, true, true );
  settextstyle( BOLD_FONT, HORIZ_DIR, 2 );
  setcolor( BLACK );
  setbkcolor( WHITE );

  // An array of five dice values, all initialized to zero.
  int dice[ 5 ] = { 0, 0, 0, 0, 0 };
  // An array indicating whether each die is currently being held.
  bool hold[ 5 ] = { false, false, false, false, false };
  // An array of scores, all initialized to -1. It is possible for a category to
  // have a value of zero, so -1 indicates the category has not yet been used.
  int scores[ 13 ] = { -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1 };

  int x, y;       // Coordinates of each mouse click.
  int rolls = 0;  // Each turn can be up to three rolls.
  int turns = 0;  // There are exactly 13 turns in a game of Yahtzee.
  while( turns < 13 )
  {
    // These lines of code control the BGI double-buffering feature.
    page = page * -1 + 1;        // The value of the page variable alternates between 0 and 1.
    setactivepage( page );       // This sets the page on which all drawing will occur.
    cleardevice();               // When drawing on a new page, clear it first.
    drawDice( dice, hold );      // Draws the dice and also indicates if each die is held.
    drawScores( scores );        // Draws the score categories and score values.
    drawButton( turns, rolls );  // Draws the button and a message to the user.
    setvisualpage( page );       // Make the current state of the game visible.

    waitForMouseClick( x, y );

    // See if the click was on a die.
    if( x < DIE_SIZE + MARGIN && y < DIE_SIZE * 5 + MARGIN * 6 )
    {
      // Hold dice if necessary.
      if( rolls > 0 && rolls < 3 )
      {
        int die = y / ( DIE_SIZE + MARGIN );
        hold[ die ] = !hold[ die ];
      }
    }
    // See if the click was on a score box.
    else if( x > WIDTH - DIE_SIZE - MARGIN && y < DIE_SIZE * 5 + MARGIN * 6 )
    {
      if( rolls > 0 )
      {
        // Box size is the total height of the dice plus margins divided by 13.
        int boxSize = ( DIE_SIZE * 5 + MARGIN * 6 ) / 13;
        int category = y / boxSize;
        // Make sure the category is unused.
        if( scores[ category ] == -1 )
        {
          // Record the score and reset the dice for the next turn.
          scores[ category ] = getScore( dice, category );
          resetDice( dice, hold );
          rolls = 0;
          ++turns;
        }
      }
    }
    // See if the click was on the button.
    else if( x > WIDTH / 2 - DIE_SIZE && x < WIDTH / 2 + DIE_SIZE &&
             y > HEIGHT - DIE_SIZE - MARGIN && y < HEIGHT - DIE_SIZE / 2 )
    {
      rollDice( dice, hold );
      ++rolls;
    }
    // All other clicks are ignored.
  }

  // Game is over, so only draw dice and the scores.
  page = page * -1 + 1;        // The value of the page variable alternates between 0 and 1.
  setactivepage( page );       // This sets the page on which all drawing will occur.
  cleardevice();               // When drawing on a new page, clear it first.
  drawDice( dice, hold );      // Draws the dice and also indicates if each die is held.
  drawScores( scores );        // Draws the score categories and score values.

  // Draw the final score in the middle of the bottom.
  settextjustify( CENTER_TEXT, CENTER_TEXT );
  char scoreString[ 18 ];
  sprintf_s( scoreString, 18, "Final Score: %d", getFinalScore( scores ) );
  outtextxy( WIDTH / 2, HEIGHT - DIE_SIZE / 2, scoreString );

  setvisualpage( page );       // Make the current state of the game visible.
  
  // Wait for a mouse click, then close the graphics window and exit the program.
  waitForMouseClick( x, y );
  closegraph();
  return EXIT_SUCCESS;
}

/*
 * This function calculates the value of the dice for a given category.
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                category is an integer in the range [0-12].
 * Post-condition: The score of the dice in that category is returned.
 *
 * NOTE: The dice array will be modified when it is sorted, but that's ok because
 *       as soon as this function is finished, it will be reset to all zeros.
 */
int getScore( int dice[], const int category )
{
  // Sorting the dice makes checking for each possibility exponentially easier.
  sort( dice );

  // Categories 0-5 correspond to Ones through Sixes
  if( category <= 5 )
  {
    // Yes, it's awkward to have the "ones" score in array location zero ... get used to it!  :)
    return sumOnly( dice, category + 1 );
  }
  else if( category == 6 && isThreeOfAKind( dice ) || category == 7 && isFourOfAKind( dice ) )
  {
    return sumAll( dice );
  }
  else if( category == 8 && isFullHouse( dice ) )
  {
    return 25;
  }
  else if( category == 9 && ( isSmallStraight( dice ) || isYahtzee( dice ) ) )
  {
    return 30;
  }
  else if( category == 10 && ( isLargeStraight( dice ) || isYahtzee( dice ) ) )
  {
    return 40;
  }
  else if( category == 11 && isYahtzee( dice ) )
  {
    return 50;
  }
  else if( category == 12 )  // Any dice roll can be counted in the Chance category.
  {
    return sumAll( dice );
  }
  else
  {
    return 0;
  }
}

/*
 * Sorts the dice into ascending order.
 * 
 * Pre-condition: dice is a valid integer array with exactly 5 elements.
 * Post-condition: The dice values are sorted in ascending order.
 */

// Code from http://http://eecs.mines.edu/Courses/csci261/images/ArrayDemo.png 
// Bubble Sort
void sort( int dice[] )
{
  /******** INSERT CODE HERE ********/
  bool swappedOne = true;
  int N = 5;
  for( int pass = 1; pass < N && swappedOne; pass++ )
  {
	  swappedOne = false;

	  for( int index = 0; index < N - pass; index++ )
	  {
		  if( dice[ index ] > dice[ index + 1 ] )
		  {
			  int temp = dice[ index ];
			  dice[ index ] = dice[ index + 1 ];
			  dice[ index ] = temp;
			  swappedOne = true; 
		  }
	  }
  }
	
}

/*
 * This function calculates the sum of all five dice values.
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements.
 * Post-condition: The sum of all dice values is returned.
 */
int sumAll( const int dice[] )
{
  /******** INSERT CODE HERE ********/
  return 0;
}

/*
 * This function calculates the sum of all dice that match the given value.
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                value is an integer in the range [1-6].
 * Post-condition: The sum of all dice that match value is returned.
 */
int sumOnly( const int dice[], const int value )
{
  /******** INSERT CODE HERE ********/
  return 0;
}

/*
 * This function determines if the dice contain three of a kind.
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                the values in the dice array are sorted in ascending order.
 * Post-condition: Returns true if the dice contain three of a kind; false otherwise.
 */
bool isThreeOfAKind( const int dice[] )
{
  /******** INSERT CODE HERE ********/
  return ( dice[ 0 ] == dice[ 1 ] == dice[ 2 ]) ||
		 ( dice[ 1 ] == dice[ 2 ] == dice[ 3 ]) ||
		 ( dice[ 2 ] == dice[ 3 ] == dice[ 4 ]);
  
  return false;
}

/*
 * This function determines if the dice contain four of a kind.
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                the values in the dice array are sorted in ascending order.
 * Post-condition: Returns true if the dice contain four of a kind; false otherwise.
 */
bool isFourOfAKind( const int dice[] )
{
  /******** INSERT CODE HERE ********/
  return ( dice[ 0 ] == dice[ 1 ] && dice[ 1 ] == dice[ 2 ] && dice[ 2 ] == dice[ 3 ] ) ||
		 ( dice[ 1 ] == dice[ 2 ] && dice[ 2 ] == dice[ 3 ] && dice[ 3 ] == dice[ 4 ] );
  return false;
  
}

/*
 * This function determines if the dice contain a full house.
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                the values in the dice array are sorted in ascending order.
 * Post-condition: Returns true if the dice contain a full house; false otherwise.
 */
bool isFullHouse( const int dice[] )
{
  /******** INSERT CODE HERE ********/
  return ( dice[ 0 ] == dice[ 1 ]) && ( dice[ 2 ] == dice[ 3 ] == dice[ 4 ]) ||
		 ( dice[ 0 ] == dice[ 1 ] == dice[ 2 ]) && ( dice[ 3 ] == dice[ 4 ]);

  return false;
}

/*
 * This function determines if the dice contain a small straight.
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                the values in the dice array are sorted in ascending order.
 * Post-condition: Returns true if the dice contain a small straight; false otherwise.
 */
bool isSmallStraight( const int dice[] )
{
  /******** INSERT CODE HERE ********/

  
  return false;
}

/*
 * This function determines if the dice contain a large straight.
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                the values in the dice array are sorted in ascending order.
 * Post-condition: Returns true if the dice contain a large straight; false otherwise.
 */
bool isLargeStraight( const int dice[] )
{
  /******** INSERT CODE HERE ********/
  return (  (( dice[ 0 ])  ==  ( dice[ 1 ] - 1 )) && (( dice[ 1 ] ) == ( dice[ 2 ] - 1 )) &&
		 (( dice[ 2 ]) == ( dice[ 3 ] - 1)) && (( dice[ 3 ]) == ( dice[ 4 ] - 1 )) ); 

  return false;
}

/*
 * This function determines if the dice contain a Yahtzee.
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                the values in the dice array are sorted in ascending order.
 * Post-condition: Returns true if the dice contain a yahtzee; false otherwise.
 */
bool isYahtzee( const int dice[] )
{
  /******** INSERT CODE HERE ********/
  return ( dice[ 0 ] == dice[ 1 ] == dice[ 2 ] == dice[ 3 ] == dice[ 4 ] ) ;

  return false;
}

/*
 * This function calculates the final score for the game.
 *
 * The final score in Yahtzee is the sum of each individual score plus a bonus of
 * 35 points if the total of the "upper" section of the scorecard is at least 63.
 * The "upper" section is the Ones, Two, Threes, Fours, Fives, and Sixes categories.
 *
 * Pre-condition: scores is a valid integer array with exactly 13 elements.
 * Post-condition: The score of the dice in that category is returned.
 */
int getFinalScore( const int scores[] )
{
  /******** INSERT CODE HERE ********/
  return 0;
}

/*
 * This function resets the dice and the hold arrays for a new turn.
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: dice and hold are valid integer array with exactly 5 elements;
 * Post-condition: The dice values are all zero; the hold values are all false.
 */
void resetDice( int dice[], bool hold[] )
{
  for( int index = 0; index < 5; index++ )
  {
    dice[ index ] = 0;
    hold[ index ] = false;
  }
}

/*
 * This function rolls the dice not marked as held.
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: dice and hold are valid integer array with exactly 5 elements;
 * Post-condition: The dice values not marked as held contain new random values.
 */
void rollDice( int dice[], const bool hold[] )
{
  for( int index = 0; index < 5; index++ )
  {
    if( !hold[ index ] )
    {
      dice[ index ] = randomBetween( 1, 6 );
    }
  }
}

/*
 * This function draws all of the dice on the left side of the graphics window.
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: dice and hold are valid integer array with exactly 5 elements;
 */
void drawDice( const int dice[], const bool hold[] )
{
  for( int index = 0; index < 5; index++ )
  {
    drawDie( dice[ index ], hold[ index ], MARGIN, MARGIN + index * ( DIE_SIZE + MARGIN ) );
  }
}

/*
 * This function draws a single die at the indicated location.
 * If the hold parameter is true, it will also draw the word HOLD.
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: pips is an integer value in the range [0-6];
 *                hold is a boolean value;
 *                x,y is a coordinate on the graphics window.
 */
void drawDie( const int pips, const bool hold, const int x, const int y )
{
  char filename[ 13 ];
  sprintf_s( filename, 13, "images/%d.jpg", pips );
  readimagefile( filename, x, y, x + DIE_SIZE, y + DIE_SIZE );
  if( hold )
  {
    int h = textheight( "X" );
    outtextxy( x + DIE_SIZE + MARGIN, y + MARGIN + h * 1, "H" );
    outtextxy( x + DIE_SIZE + MARGIN, y + MARGIN + h * 2, "O" );
    outtextxy( x + DIE_SIZE + MARGIN, y + MARGIN + h * 3, "L" );
    outtextxy( x + DIE_SIZE + MARGIN, y + MARGIN + h * 4, "D" );
  }
}

/*
 * This function draws the score categories and score values in the graphics window.
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: scores is a valid integer array with exactly 13 elements.
 */
void drawScores( const int scores[] )
{
  // This is an array of character strings, which we will finally discuss next week!
  char* categories[] = { "Ones", "Twos", "Threes", "Fours", "Fives", "Sixes",
                         "Three of a Kind", "Four of a Kind", "Full House",
                         "Small Straight", "Large Straight", "Yahtzee", "Chance" };

  // Dimensions of the text to be drawn.
  int txtWidth  = textwidth( "XXXXXXX" );
  int txtHeight = textheight( "XXXXXXX" );
  // The x-coordinate where category labels will be right-justified and scores left-justified.
  int x = WIDTH - DIE_SIZE;
  // The starting y-coordinate for the top category.
  int y = MARGIN;
  // This delta-y evenly spaces the 13 categories in the same veritcal space as five dice, plus margins.
  int dy = ( DIE_SIZE * 5 + MARGIN * 6 ) / 13;

  // Draw the category labels, right justified at the x-coordinate.
  settextjustify( RIGHT_TEXT, TOP_TEXT );
  for( int index = 0; index < 13; index++, y += dy )
  {
    outtextxy( x, y, categories[ index ] );
  }

  // Draw the scores, right justified at the x-coordinate.
  settextjustify( LEFT_TEXT, TOP_TEXT );
  // Re-start at the same y-coordinate.
  y = MARGIN;
  // Use to make a character string out of the integer score value.
  char scoreString[ 6 ];
  for( int index = 0; index < 13; index++, y += dy )
  {
    // Scores are initialized to -1 to indicate an unused category.
    if( scores[ index ] >= 0 )
    {
      // Make a character string with the integer score and display it.
      sprintf_s( scoreString, 6, "%5d", scores[ index ] );
      outtextxy( x, y, scoreString );
    }
    // Draw a box around the score.
    rectangle( x + 2, y - 2, x + txtWidth, y + txtHeight + 2 );
  }
}

/*
 * This function draws the button and a message at the bottom of the screen.
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: turns is an integer value in the range [0-13];
 *                rolls is an integer value in the range [0-3].
 */
void drawButton( const int turns, const int rolls )
{
  settextjustify( CENTER_TEXT, BOTTOM_TEXT );
  if( turns == 13 )
  {
    outtextxy( WIDTH / 2, HEIGHT - MARGIN, "Game over." );
  }
  else if( rolls == 0 )
  {
    outtextxy( WIDTH / 2, HEIGHT - MARGIN, "Click the Roll Dice button." );
  }
  else if( rolls < 3 )
  {
    outtextxy( WIDTH / 2, HEIGHT - MARGIN, "Click dice to hold, then roll." );
  }
  else
  {
    outtextxy( WIDTH / 2, HEIGHT - MARGIN, "Click scorecard to record this turn." );
  }

  if( rolls < 3 && turns < 13 )
  {
    readimagefile( "images/roll.jpg", WIDTH / 2 - DIE_SIZE, HEIGHT - DIE_SIZE - MARGIN, WIDTH / 2 + DIE_SIZE, HEIGHT - DIE_SIZE / 2 );
  }
  else
  {
    readimagefile( "images/button.jpg", WIDTH / 2 - DIE_SIZE, HEIGHT - DIE_SIZE - MARGIN, WIDTH / 2 + DIE_SIZE, HEIGHT - DIE_SIZE / 2 );
  }
}

/*
 * This function waits for a mouse click and when a click occurs, it puts the x,y
 * coordinate of the click into the x and y locations passed as reference parameters.
 * (We will discuss reference parameters later in the semester!)
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: dice is a valid integer array with exactly 5 elements;
 *                category is an integer in the range [0-12].
 * Post-condition: The score of the dice in that category is returned.
 */
void waitForMouseClick( int &x, int &y )
{
  while( !ismouseclick( WM_LBUTTONDOWN ) )
  {
    delay( 250 );
  }
  getmouseclick( WM_LBUTTONDOWN, x, y );
}

/*
 * This function returns a random integer between lowerBound and upperBound, inclusive.
 *
 *                ***********************************
 *                *** DO NOT MODIFY THIS FUNCTION ***
 *                ***********************************
 *
 * Pre-condition: The input values satisfy the condition: lowerBound <= upperBound
 * Post-condition: The value returned satisfies the condition: lowerBound <= number <= upperBound
 */
int randomBetween( const int lowerBound, const int upperBound )
{
  if( upperBound < lowerBound )
  {
    cerr << "ERROR: Parameters to randomBetween do not satisfy condition lowerBound <= upperBound." << endl;
    return 0;
  }
  else
  {
    return rand() % ( upperBound - lowerBound + 1 ) + lowerBound;
  }
}

