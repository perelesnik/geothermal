// firmware for heat pump controller

#include "U8glib.h"
#include <TimerOne.h>
#include <Button.h>
#include <EEPROM.h>
#include <TimeLib.h>
#include <DS1307RTC.h>
//#include <Stepper_28BYJ.h>

//#define STEPS 4078

//Stepper_28BYJ stepper(STEPS, 8, 9, 10, 11);


tmElements_t tm;
int Hour, Minute, Second, Day, Month, Year;
int daytimeBegin, daytimeEnd, predaytimeBegin, predaytimeEnd;
int economyAmount, preeconomyAmount;
bool daytimeOn, economyMenu, daybegsetflag, deyendsetflag, economysetflag, ecosaveflag;

bool GVSmenuFlag = false;
byte settherm = 60; // установка целевой температуры отопления
byte setTimedTherm = 60; // установка целевой температуры отопления

byte GVSsettherm = 60; // установка целевой температуры гвс
byte GVSsetTimedTherm = 60; // установка целевой температуры гвс
float GVSrealtherm;
float GVSdeltatherm = 3;


float realtherm;
float deltatherm = 3;
bool thermmenuflag = LOW;
bool deltathermflag = LOW;
bool flag_work = false;

byte up_limit = 61;
byte podmes_delta = 2;
byte low_podmes_delta = 8;

byte presettherm = 60; // установка температуры в главном контроллере
byte predeltatherm = 60;
byte GVSpresettherm = 60; // установка температуры в главном контроллере
byte GVSpredeltatherm = 60;

int submenu = 0;
int prepos = 0; // предыдущее положение
bool obnulpos; // обнуление позиции при входе в меню
bool LCDflag = LOW; // флаг изменения значения подсветки
byte preLCDled = 120; // предыдущее занчение подсветки
int preposLED = 0; // обнудяем энкодер для считывания значения подсветки
byte preposLED_on = 10; // счетчик для включения подсветки
bool LED_on = true;

int error_code = 0;

// таймеры задержек
unsigned long worktime; // счетчик времени
unsigned long currentTime; // таймер начала работы

unsigned long error_1_time; // задержка времени определения ошибки
unsigned long error_2_time;
unsigned long error_3_time;
unsigned long error_4_time;
unsigned long error_5_time;

int errordelay = 5000; // значение задержки времени определения ошибки

unsigned long encodertime;

unsigned long presentTime;

unsigned long total_delay;

unsigned long total_start_time;
unsigned long total_start_period = 180000;
//unsigned long total_start_period = 10000;
unsigned long LED_time;
unsigned long LED_delay = 120000;

int buttonSTART_STOPperiod = 1500; // время длительного нажатия
unsigned long START_STOPtime;
bool longClick = false;
bool pauseON = false;
bool sensScreenOn = false;
bool enginMenuOn = false;
bool enginChangeFlag = false;

unsigned long sensorstime;
int sensorsperiod = 1000;

unsigned long timer_1_rel;
int timer_1_delay = 5000;

unsigned long timer_2_rel;
int timer_2_delay = 2000;

unsigned long timer_3_rel;
int timer_3_delay = 2000;

unsigned long timer_4_rel;
int timer_4_delay = 1000;


unsigned long outscreentime;
unsigned long outscreenperiod = 30000;
boolean outscreenOn;

// это для энкодера
#define Butt 2 // пин кнопки
Button encoderA (3, 4); // сигнал A
Button encoderB (4, 4); //  сигнал B
int pos = 0; // пооложение энкодера
int menu = 0; // входим в меню по такту кнопки
bool buttflag = LOW; // перебираем положение кнопки

U8GLIB_ST7920_128X64 u8g(45, 43, 42, U8G_PIN_NONE); //иннициализация дисплея


int speakerPin = 5;  // выход на динамик
int LCDledPin = 44;    // Подключение подсветки дисплея

// номера контактов выходов на твердотельные реле
int rel1 = 32;
int rel2 = 33;
int rel3 = 34;
int rel4 = 35;
int rel5 = 36;
int rel6 = 37;
int rel7 = 38;
int rel8 = 39;

int RD_pin = 18; //контакт РД
int RDsupply_pin = 19;

#define B 3950 // B-коэффициент
#define SERIAL_R 10000 // сопротивление последовательного резистора, 10 кОм
#define THERMISTOR_R 10000 // номинальное сопротивления термистора, 10 кОм
#define NOMINAL_T 25 // номинальная температура (при которой TR = 10 кОм)

//номера контактов входов термодатчиков
const byte tempPin1 = A1;
const byte tempPin2 = A2;
const byte tempPin3 = A3;
const byte tempPin4 = A4;
const byte tempPin5 = A5;
const byte tempPin6 = A6;
const byte tempPin7 = A7;
const byte tempPin8 = A8;
const byte tempPin9 = A9;

// переменные для хранения данных термодатчиков
float in_temp11, in_temp12, in_temp21, in_temp22, in_temp31,
      in_temp32, in_temp41, in_temp42, in_temp51, in_temp52;

// распределенные переменные подача - обратка
float r_temp11, r_temp12, r_temp21, r_temp22, r_temp31,
      r_temp32, r_temp41, r_temp42, r_temp51, r_temp52;

// дельта тепмературы на парах датчивов

float d_temp1, d_temp2, d_temp3, d_temp4, d_temp5;

// поправки датчиков

float adj_temp11, adj_temp12, adj_temp21, adj_temp22, adj_temp31;
float adj_temp32, adj_temp41, adj_temp42, adj_temp51, adj_temp52;

// наличие датчиков

boolean td11, td21, td31, td41, td51;
boolean td12, td22, td32, td42, td52;

// флаги включения реле

boolean trel1, trel2, trel3, trel4, trel5, trel6, trel7, trel8; // задача
boolean must_to_stop, tstop, stoptrel1, stoptrel2, stoptrel3, stoptrel4, stoptrel5, stoptrel6, stoptrel7, stoptrel8; // задача
boolean frel1, frel2, frel3, frel4, frel5, frel6, frel7, frel8; // факт исполнения

// значения заданных температур

byte st1, st2, st3, st4, st5;

// дельты

byte delta1, delta2, delta3, delta4, delta5;

// номер отслеживаемых датчиков

byte targetnumber;


byte LCDled = 80;  // значение подсветки

void(* resetFunc) (void) = 0;

void Button() {
  presentTime = millis();
  if (presentTime - encodertime >= 10) {
    encodertime = presentTime;
    if (digitalRead(Butt) == LOW && buttflag == LOW) { //проверка нажатия на кнопку
      menu++;
      tone(speakerPin, 800, 200);
      if (menu >= 3 && !economyMenu) {
        st3 = GVSsettherm;
        delta3 = GVSdeltatherm;
        if (EEPROM.read(0) != LCDled) {
          EEPROM.write(0, LCDled);
        }
        if (EEPROM.read(1) != settherm) {
          EEPROM.write(1, settherm);
        }
        if (EEPROM.read(2) != deltatherm) {
          EEPROM.write(2, deltatherm);
        }
        if (EEPROM.read(3) != targetnumber) {
          EEPROM.write(3, targetnumber);
        }
        if (EEPROM.read(11) != st1) {
          EEPROM.write(11, st1);
        }
        if (EEPROM.read(12) != st2) {
          EEPROM.write(12, st2);
        }
        if (EEPROM.read(13) != st3) {
          EEPROM.write(13, st3);
        }
        if (EEPROM.read(14) != st4) {
          EEPROM.write(14, st4);
        }
        if (EEPROM.read(15) != st5) {
          EEPROM.write(15, st5);
        }
        if (EEPROM.read(21) != delta1) {
          EEPROM.write(21, delta1);
        }
        if (EEPROM.read(22) != delta2) {
          EEPROM.write(22, delta2);
        }
        if (EEPROM.read(23) != delta3) {
          EEPROM.write(23, delta3);
        }
        if (EEPROM.read(24) != delta4) {
          EEPROM.write(24, delta4);
        }
        if (EEPROM.read(25) != delta5) {
          EEPROM.write(25, delta5);
        }

        menu = 0;
        pos = 1;
      }
      if (menu == 1) {
        pos = 1;
      }
      buttflag = HIGH;
    }
  }
  if (digitalRead(Butt) == HIGH) { //проверка нажатия на кнопку
    buttflag = LOW;
  }

}

// обработчик прерывания 250 мкс
void timerInterrupt() {
  encoderA.filterAvarage(); // вызов метода фильтрации
  encoderB.filterAvarage(); // вызов метода фильтрации
  if ( !pauseON) {
    if ( encoderA.flagClick == true ) {
      encoderA.flagClick = false;
      if ( encoderB.flagPress == false) {
        // против часовой стрелки
        pos--;
        tone(speakerPin, 1100, 100);
        if (pos < 1 && menu == 0 && !enginMenuOn) {
          pos = 1;
        }
        if (pos < 0 && menu == 0 && enginMenuOn) {
          pos = 0;
        }

      }
      else {
        // по часовой стрелке
        pos++;
        tone(speakerPin, 1400, 100);

        if (pos > 4 && menu == 0 && !enginMenuOn) {
          pos = 4;
        }
      }
    }
  }
}

void readsensors() {

  int t1 = analogRead( tempPin1 );
  float tr1 = 1023.0 / t1 - 1;
  tr1 = SERIAL_R / tr1;

  in_temp11 = tr1 / THERMISTOR_R; // (R/Ro)
  in_temp11 = log(in_temp11); // ln(R/Ro)
  in_temp11 /= B; // 1/B * ln(R/Ro)
  in_temp11 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp11 = 1.0 / in_temp11; // Invert
  in_temp11 -= 273.15;

  int t2 = analogRead( tempPin2 );
  float tr2 = 1023.0 / t2 - 1;
  tr2 = SERIAL_R / tr2;

  in_temp12 = tr2 / THERMISTOR_R; // (R/Ro)
  in_temp12 = log(in_temp12); // ln(R/Ro)
  in_temp12 /= B; // 1/B * ln(R/Ro)
  in_temp12 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp12 = 1.0 / in_temp12; // Invert
  in_temp12 -= 273.15;


  if (in_temp11 > -40 && in_temp11 < 90) {
    td11 = true;
    error_1_time = presentTime;
  }
  else {
    td11 = false;
    Serial.println("T1_(1)= not connected");
  }

  if (in_temp12 > -40 && in_temp12 < 90) {
    td12 = true;
    // error_1_time = presentTime;
  }
  else {
    td12 = false;
    Serial.println("T1_(2)= not connected");
  }

  if (td11 && td12) {
    //  if (in_temp11 >= in_temp12) {
    r_temp11 = in_temp11 + adj_temp11;
    r_temp12 = in_temp12 + adj_temp12;
    //    }
    //    else {
    //      r_temp11 = in_temp12;
    //      r_temp12 = in_temp11;
    //    }
    d_temp1 = r_temp11 - r_temp12;
  }
  else if (td11) {
    r_temp11 = in_temp11 + adj_temp11;
    r_temp12 = 0;
  }
  if (!td11) {
    r_temp11 = 3;
  }
  //////////////////////////
  int t3 = analogRead( tempPin3 );
  float tr3 = 1023.0 / t3 - 1;
  tr3 = SERIAL_R / tr3;

  in_temp21 = tr3 / THERMISTOR_R; // (R/Ro)
  in_temp21 = log(in_temp21); // ln(R/Ro)
  in_temp21 /= B; // 1/B * ln(R/Ro)
  in_temp21 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp21 = 1.0 / in_temp21; // Invert
  in_temp21 -= 273.15;

  int t4 = analogRead( tempPin4 );
  float tr4 = 1023.0 / t4 - 1;
  tr4 = SERIAL_R / tr4;

  in_temp22 = tr4 / THERMISTOR_R; // (R/Ro)
  in_temp22 = log(in_temp22); // ln(R/Ro)
  in_temp22 /= B; // 1/B * ln(R/Ro)
  in_temp22 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp22 = 1.0 / in_temp22; // Invert
  in_temp22 -= 273.15;

  if (in_temp21 > -40 && in_temp21 < 90) {
    td21 = true;
    error_2_time = presentTime;
  }
  else {
    td21 = false;
    Serial.println("T2_(1)= not connected");
  }

  if (in_temp22 > -40 && in_temp22 < 90) {
    td22 = true;
    // error_2_time = presentTime;
  }
  else {
    td22 = false;
    Serial.println("T2_(2)= not connected");
  }

  if (td21 && td22) {
    //  if (in_temp21 >= in_temp22) {
    r_temp21 = in_temp21 + adj_temp21;
    r_temp22 = in_temp22 + adj_temp22;
    //    }
    //    else {
    //      r_temp21 = in_temp22;
    //      r_temp22 = in_temp21;
    //    }
    d_temp2 = r_temp21 - r_temp22;
  }
  else if (td21) {
    r_temp21 = in_temp21 + adj_temp21;
    r_temp22 = 0;
  }

  ////////////////////////////////
  int t5 = analogRead( tempPin5 );
  float tr5 = 1023.0 / t5 - 1;
  tr5 = SERIAL_R / tr5;

  in_temp31 = tr5 / THERMISTOR_R; // (R/Ro)
  in_temp31 = log(in_temp31); // ln(R/Ro)
  in_temp31 /= B; // 1/B * ln(R/Ro)
  in_temp31 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp31 = 1.0 / in_temp31; // Invert
  in_temp31 -= 273.15;

  int t6 = analogRead( tempPin6 );
  float tr6 = 1023.0 / t6 - 1;
  tr6 = SERIAL_R / tr6;

  in_temp32 = tr6 / THERMISTOR_R; // (R/Ro)
  in_temp32 = log(in_temp32); // ln(R/Ro)
  in_temp32 /= B; // 1/B * ln(R/Ro)
  in_temp32 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp32 = 1.0 / in_temp32; // Invert
  in_temp32 -= 273.15;


  if (in_temp31 > -40 && in_temp31 < 90) {
    td31 = true;
    error_3_time = presentTime;
  }
  else {
    td31 = false;
    Serial.println("T3_(1)= not connected");
  }

  if (in_temp32 > -40 && in_temp32 < 90) {
    td32 = true;
    // error_3_time = presentTime;
  }
  else {
    td32 = false;
    Serial.println("T3_(2)= not connected");
  }

  if (td31 && td32) {
    //   if (in_temp31 >= in_temp32) {
    r_temp31 = in_temp31 + adj_temp31;
    r_temp32 = in_temp32 + adj_temp32;
    //    }
    //    else {
    //      r_temp31 = in_temp32;
    //      r_temp32 = in_temp31;
    //    }
    d_temp3 = r_temp31 - r_temp32;
  }
  else if (td31) {
    r_temp31 = in_temp31 + adj_temp31;
    r_temp32 = 0;
  }
  if (!td31) {
    r_temp31 = 0;
  }
  ////////////////////////////////
  int t7 = analogRead( tempPin7 );
  float tr7 = 1023.0 / t7 - 1;
  tr7 = SERIAL_R / tr7;

  in_temp41 = tr7 / THERMISTOR_R; // (R/Ro)
  in_temp41 = log(in_temp41); // ln(R/Ro)
  in_temp41 /= B; // 1/B * ln(R/Ro)
  in_temp41 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp41 = 1.0 / in_temp41; // Invert
  in_temp41 -= 273.15;

  int t8 = analogRead( tempPin8 );
  float tr8 = 1023.0 / t8 - 1;
  tr8 = SERIAL_R / tr8;

  in_temp42 = tr8 / THERMISTOR_R; // (R/Ro)
  in_temp42 = log(in_temp42); // ln(R/Ro)
  in_temp42 /= B; // 1/B * ln(R/Ro)
  in_temp42 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp42 = 1.0 / in_temp42; // Invert
  in_temp42 -= 273.15;


  if (in_temp41 > -40 && in_temp41 < 90) {
    td41 = true;
    error_4_time = presentTime;
  }
  else {
    td41 = false;
    Serial.println("T3_(1)= not connected");
  }

  if (in_temp42 > -40 && in_temp42 < 90) {
    td42 = true;
    // error_4_time = presentTime;
  }
  else {
    td42 = false;
    Serial.println("T3_(2)= not connected");
  }

  if (td41 && td42) {
    //  if (in_temp41 >= in_temp42) {
    r_temp41 = in_temp41 + adj_temp41;
    r_temp42 = in_temp42 + adj_temp42;
    //    }
    //    else {
    //      r_temp41 = in_temp42;
    //      r_temp42 = in_temp41;
    //    }
    d_temp4 = r_temp41 - r_temp42;
  }
  else if (td41) {
    r_temp41 = in_temp41 + adj_temp41;
    r_temp42 = 0;
  }
  ////////////////////////////////
  int t9 = analogRead( tempPin9 );
  float tr9 = 1023.0 / t9 - 1;
  tr9 = SERIAL_R / tr9;

  // Serial.println (t9);

  in_temp51 = tr9 / THERMISTOR_R; // (R/Ro)
  in_temp51 = log(in_temp51); // ln(R/Ro)
  in_temp51 /= B; // 1/B * ln(R/Ro)
  in_temp51 += 1.0 / (NOMINAL_T + 273.15); // + (1/To)
  in_temp51 = 1.0 / in_temp51; // Invert
  in_temp51 -= 273.15;


  if (in_temp51 > -40 && in_temp51 < 90) {
    td51 = true;
    error_5_time = presentTime;

    r_temp51 = in_temp51 + adj_temp51;
    r_temp52 = 0;
    d_temp5 = r_temp51 - r_temp52;
  }
  else {
    td51 = false;
    Serial.println("T5_(1)= not connected");
  }
}

void adj_eeprom_write() {
  if (EEPROM.read(111) != (adj_temp11 * 5) + 100) {
    EEPROM.write(111, (adj_temp11 * 5) + 100);
  }
  if (EEPROM.read(112) != (adj_temp12 * 5) + 100) {
    EEPROM.write(112, (adj_temp12 * 5) + 100);
  }
  if (EEPROM.read(121) != (adj_temp21 * 5) + 100) {
    EEPROM.write(121, (adj_temp21 * 5) + 100);
  }
  if (EEPROM.read(122) != (adj_temp22 * 5) + 100) {
    EEPROM.write(122, (adj_temp22 * 5) + 100);
  }
  if (EEPROM.read(131) != (adj_temp31 * 5) + 100) {
    EEPROM.write(131, (adj_temp31 * 5) + 100);
  }
  if (EEPROM.read(132) != (adj_temp32 * 5) + 100) {
    EEPROM.write(132, (adj_temp32 * 5) + 100);
  }
  if (EEPROM.read(141) != (adj_temp41 * 5) + 100) {
    EEPROM.write(141, (adj_temp41 * 5) + 100);
  }
  if (EEPROM.read(142) != (adj_temp42 * 5) + 100) {
    EEPROM.write(142, (adj_temp42 * 5) + 100);
  }
  if (EEPROM.read(151) != (adj_temp51 * 5) + 100) {
    EEPROM.write(151, (adj_temp51 * 5) + 100);
  }
  if (EEPROM.read(152) != (adj_temp52 * 5) + 100) {
    EEPROM.write(152, (adj_temp52 * 5) + 100);
  }
}

void igeneer_menudraw() {

  if (menu == 0) {
    submenu = pos;
  }
  if (menu == 0 && submenu > 10) {
    pos = 10;
  }
  if (submenu < 6) {
    if (!enginChangeFlag) {
      u8g.drawBox(0, submenu * 10 + 3 , 5, 8);
    }
    else {
      u8g.drawLine(0, submenu * 10 + 11 , 50, submenu * 10 + 11);
    }
  }
  if (submenu >= 6) {
    if (!enginChangeFlag) {
      u8g.drawBox(58, submenu * 10 - 60 + 14 , 5, 8);
    }
    else {
      u8g.drawLine(58, submenu * 10 + 22 - 60 , 100, submenu * 10 + 22 - 60);
    }
  }
  u8g.setFont(u8g_font_4x6r);
  u8g.setPrintPos(0, 8);
  u8g.print(pos);
  u8g.setPrintPos(10, 8);
  u8g.print( "BACK");

  if (menu >= 1 && submenu == 0) {
    adj_eeprom_write();
    sensScreenOn = false;
    enginMenuOn = false;
    longClick = false;
    pauseON = false;
    enginChangeFlag = false;
    menu = 0;
    pos = 1;
  }

  if (menu == 1 && submenu > 0 && !enginChangeFlag) {
    pos = 0;
    prepos = pos;
    enginChangeFlag = true;
  }

  if (menu >= 2 && submenu > 0 && enginChangeFlag) {
    menu = 0;
    pos = submenu;
    adj_eeprom_write();
    enginChangeFlag = false;
  }


  u8g.setPrintPos(8, 20);
  if (td11) {
    u8g.print( "T1(1)=" + String(r_temp11));
  }
  else {
    u8g.print( "T1(1)=NC");
  }
  u8g.setPrintPos(62, 20);
  if (td12) {
    u8g.print( " (2)=" + String(r_temp12));
  }
  else {
    u8g.print( " T1(2)=NC");
  }

  u8g.setPrintPos(8, 30);
  if (td21) {
    u8g.print( "T2(1)=" + String(r_temp21));
  }
  else {
    u8g.print( "T2(1)=NC");
  }
  u8g.setPrintPos(62, 30);
  if (td22) {
    u8g.print( " (2)=" + String(r_temp22));
  }
  else {
    u8g.print( " T2(2)=NC");
  }

  u8g.setPrintPos(8, 40);
  if (td31) {
    u8g.print( "T3(1)=" + String(r_temp31));
  }
  else {
    u8g.print( "T3(1)=NC");
  }
  u8g.setPrintPos(62, 40);
  if (td32) {
    u8g.print( " (2)=" + String(r_temp32));
  }
  else {
    u8g.print( " T3(2)=NC");
  }

  u8g.setPrintPos(8, 50);
  if (td41) {
    u8g.print( "T4(1)=" + String(r_temp41));
  }
  else {
    u8g.print( "T4(1)=NC");
  }
  u8g.setPrintPos(62, 50);
  if (td42) {
    u8g.print( " (2)=" + String(r_temp42));
  }
  else {
    u8g.print( " T4(2)=NC");
  }


  u8g.setPrintPos(8, 60);
  if (td51) {
    u8g.print( "T5(1)=" + String(r_temp51));
  }
  else {
    u8g.print( "T5(1)=NC");
  }
  u8g.setPrintPos(62, 60);
  if (td52) {
    u8g.print( " (2)=" + String(r_temp52));
  }
  else {
    u8g.print( " T5(2)=NC");
  }

  u8g.setPrintPos(60, 8);
  if (submenu == 1) {
    u8g.print( "Correction= " + String(adj_temp11));
    if (menu == 1) {
      adj_temp11 = adj_temp11 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 2) {
    u8g.print( "Correction= " + String(adj_temp21));
    if (menu == 1) {
      adj_temp21 = adj_temp21 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 3) {
    u8g.print( "Correction= " + String(adj_temp31));
    if (menu == 1) {
      adj_temp31 = adj_temp31 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 4) {
    u8g.print( "Correction= " + String(adj_temp41));
    if (menu == 1) {
      adj_temp41 = adj_temp41 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 5) {
    u8g.print( "Correction= " + String(adj_temp51));
    if (menu == 1) {
      adj_temp51 = adj_temp51 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 6) {
    u8g.print( "Correction= " + String(adj_temp12));
    if (menu == 1) {
      adj_temp12 = adj_temp12 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 7) {
    u8g.print( "Correction= " + String(adj_temp22));
    if (menu == 1) {
      adj_temp22 = adj_temp22 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 8) {
    u8g.print( "Correction= " + String(adj_temp32));
    if (menu == 1) {
      adj_temp32 = adj_temp32 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 9) {
    u8g.print( "Correction= " + String(adj_temp42));
    if (menu == 1) {
      adj_temp42 = adj_temp42 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }
  if (submenu == 10) {
    u8g.print( "Correction= " + String(adj_temp52));
    if (menu == 1) {
      adj_temp52 = adj_temp52 + (pos - prepos) * 0.2;
      prepos = pos;
    }
  }

}

void sensorsdraw(void) {
  u8g.drawFrame(0, 12, 128, 52);
  u8g.setFont(u8g_font_4x6r);
  u8g.setPrintPos(0, 8);
  u8g.print(pos);
  u8g.setPrintPos(48, 8);
  u8g.print( "SENSORS:");
  u8g.setPrintPos(114, 8);
  u8g.print( (outscreenperiod - (presentTime - outscreentime)) / 1000);

  u8g.setPrintPos(3, 20);
  if (td11) {
    u8g.print( "T1(1)=" + String(r_temp11));
  }
  else {
    u8g.print( "T1(1)=NC");
  }
  u8g.setPrintPos(48, 20);
  if (td12) {
    u8g.print( " (2)=" + String(r_temp12));
  }
  else {
    u8g.print( " T1(2)=NC");
  }
  if (td11 && td12) {
    u8g.setPrintPos(90, 20);
    u8g.print(  " D=" + String(d_temp1));
  }

  u8g.setPrintPos(3, 30);
  if (td21) {
    u8g.print( "T2(1)=" + String(r_temp21));
  }
  else {
    u8g.print( "T2(1)=NC");
  }
  u8g.setPrintPos(48, 30);
  if (td22) {
    u8g.print( " (2)=" + String(r_temp22));
  }
  else {
    u8g.print( " T2(2)=NC");
  }
  if (td21 && td22) {
    u8g.setPrintPos(90, 30);
    u8g.print(  " D=" + String(d_temp2));
  }

  u8g.setPrintPos(3, 40);
  if (td31) {
    u8g.print( "T3(1)=" + String(r_temp31));
  }
  else {
    u8g.print( "T3(1)=NC");
  }
  u8g.setPrintPos(48, 40);
  if (td32) {
    u8g.print( " (2)=" + String(r_temp32));
  }
  else {
    u8g.print( " T3(2)=NC");
  }
  if (td31 && td32) {
    u8g.setPrintPos(90, 40);
    u8g.print(  " D=" + String(d_temp3));
  }

  u8g.setPrintPos(3, 50);
  if (td41) {
    u8g.print( "T4(1)=" + String(r_temp41));
  }
  else {
    u8g.print( "T4(1)=NC");
  }
  u8g.setPrintPos(48, 50);
  if (td42) {
    u8g.print( " (2)=" + String(r_temp42));
  }
  else {
    u8g.print( " T4(2)=NC");
  }
  if (td41 && td42) {
    u8g.setPrintPos(90, 50);
    u8g.print(  " D=" + String(d_temp4));
  }

  u8g.setPrintPos(3, 60);
  if (td51) {
    u8g.print( "T5(1)=" + String(r_temp51));
  }
  else {
    u8g.print( "T5(1)=NC");
  }
  u8g.setPrintPos(48, 60);
  if (td52) {
    u8g.print( " (2)=" + String(r_temp52));
  }
  else {
    u8g.print( " T5(2)=NC");
  }
  if (td51 && td52) {
    u8g.setPrintPos(90, 60);
    u8g.print(  " D=" + String(d_temp5));
  }
}

void eepromdraw() {
  u8g.drawBox(3, targetnumber * 10 + 4 , 5, 8);
  u8g.drawFrame(0, 12, 128, 52);
  u8g.setFont(u8g_font_4x6r);
  u8g.setPrintPos(0, 8);
  u8g.print(pos);
  u8g.setPrintPos(48, 8);
  u8g.print( "MEMORY:");
  u8g.setPrintPos(114, 8);
  u8g.print( (outscreenperiod - (presentTime - outscreentime)) / 1000);
  u8g.setPrintPos(9, 20);
  u8g.print("Set(1)=" + String(settherm) + " Delta=" + String(deltatherm, 0));
  u8g.setPrintPos(9, 30);
  u8g.print("Set(2)=" + String(st2) + " Delta=" + String(delta2));
  u8g.setPrintPos(9, 40);
  u8g.print("Set(3)=" + String(st3) + " Delta=" + String(delta3));
  u8g.setPrintPos(9, 50);
  u8g.print("Set(4)=" + String(st4) + " Delta=" + String(delta4));
  u8g.setPrintPos(9, 60);
  // u8g.print("Set(5)=" + String(st5) + " Delta=" + String(delta5));
  u8g.print("Freon High Temp Limit = " + String(up_limit));
}

void releydraw() {
  u8g.drawFrame(0, 12, 128, 42);
  u8g.setFont(u8g_font_4x6r);
  u8g.setPrintPos(0, 8);
  u8g.print(pos);
  u8g.setPrintPos(48, 8);
  u8g.print( "RELEYS:");
  u8g.setPrintPos(114, 8);
  u8g.print( (outscreenperiod - (presentTime - outscreentime)) / 1000);
  u8g.setPrintPos(9, 20);
  if (frel1) {
    u8g.print( "Reley 1 = ON");
  }
  else {
    u8g.print( "Reley 1 = OFF");
  }
  u8g.setPrintPos(9, 30);
  if (frel2) {
    u8g.print( "Reley 2 = ON");
  }
  else {
    u8g.print( "Reley 2 = OFF");
  }
  u8g.setPrintPos(9, 40);
  if (frel3) {
    u8g.print( "Reley 3 = ON");
  }
  else {
    u8g.print( "Reley 3 = OFF");
  }
  u8g.setPrintPos(9, 50);
  if (frel4) {
    u8g.print( "Reley 4 = ON");
  }
  else {
    u8g.print( "Reley 4 = OFF");
  }
  u8g.setPrintPos(70, 20);
  if (frel5) {
    u8g.print( "Reley 5 = ON");
  }
  else {
    u8g.print( "Reley 5 = OFF");
  }
  u8g.setPrintPos(70, 30);
  if (frel6) {
    u8g.print( "Reley 6 = ON");
  }
  else {
    u8g.print( "Reley 6 = OFF");
  }
  u8g.setPrintPos(70, 40);
  if (frel7) {
    u8g.print( "Reley 7 = ON");
  }
  else {
    u8g.print( "Reley 7 = OFF");
  }
  u8g.setPrintPos(70, 50);
  if (frel8) {
    u8g.print( "Reley 8 = ON");
  }
  else {
    u8g.print( "Reley 8 = OFF");
  }
}

void menudraw(void) {
  u8g.setFont(u8g_font_5x7r);
  if (daytimeOn && economyAmount > 0) {
    u8g.setPrintPos(48, 7);
    u8g.print("Economy ON");
    u8g.setPrintPos(112, 32);
    u8g.print("(");
    u8g.setPrintPos(116, 32);
    u8g.print(settherm);
    u8g.setPrintPos(125, 32);
    u8g.print(")");
    u8g.setPrintPos(112, 44);
    u8g.print("(");
    u8g.setPrintPos(116, 44);
    u8g.print(settherm - deltatherm);
    u8g.setPrintPos(125, 44);
    u8g.print(")");
  }
  else {
    u8g.setPrintPos(48, 7);
    u8g.print("Economy OFF");
  }
  u8g.setPrintPos(114, 8);
  u8g.print( (outscreenperiod - (presentTime - outscreentime)) / 1000);

  if (obnulpos) {
    pos = 0;
    obnulpos = false;
  }
  u8g.drawBox(0, submenu * 12 , 5, 10);
  u8g.drawHLine(0, submenu * 12 + 10 , 120);
  u8g.setFont(u8g_font_7x14Br);
  u8g.setPrintPos(10, 10);
  if (!GVSmenuFlag) {
    u8g.print("GVS");
  }
  else {
    u8g.print("Back");
  }
  u8g.setPrintPos(10, 22);
  u8g.print("Econom");
  if (!GVSmenuFlag) {
    u8g.setPrintPos(10, 34);
    u8g.print("Set < Therm");
    u8g.setPrintPos(10, 46);
    u8g.print("Set > Therm");
  }
  else {
    u8g.setPrintPos(10, 34);
    u8g.print("Set < GVS");
    u8g.setPrintPos(10, 46);
    u8g.print("Set > GVS");
  }
  if (!GVSmenuFlag) {
    u8g.setPrintPos(10, 58);
    u8g.print("Brightness %");
    u8g.setPrintPos(98, 58);
    u8g.print(map (LCDled, 0, 255, 0, 100));
  }

  if (thermmenuflag == LOW && deltathermflag == LOW) {
    if (!GVSmenuFlag) {
      u8g.setPrintPos(98, 34);
      u8g.print(setTimedTherm);
      u8g.setPrintPos(98, 46);
      u8g.print(setTimedTherm - deltatherm, 0);
    }
    else {
      u8g.setPrintPos(98, 34);
      u8g.print(GVSsettherm);
      u8g.setPrintPos(98, 46);
      u8g.print(GVSsettherm - GVSdeltatherm, 0);
    }
    u8g.setPrintPos(58, 22);
    u8g.print(daytimeBegin);
    u8g.setPrintPos(71, 22);
    u8g.print("-");
    u8g.setPrintPos(80, 22);
    u8g.print(daytimeEnd);
    u8g.setPrintPos(93, 22);
    u8g.print(":-");
    u8g.setPrintPos(110, 22);
    u8g.print(economyAmount);
  }
  if (menu == 1) {
    submenu = pos;
    if (submenu >= 5 && !GVSmenuFlag) {
      pos = 0;
    }
    if (submenu >= 4 && GVSmenuFlag) {
      pos = 0;
    }
    if (submenu < 0) {
      pos = 0;
    }
  }

  if (submenu == 0 && menu == 2 && !GVSmenuFlag) {
    GVSmenuFlag = true;
    menu = 1;
    pos = 0;
    submenu = 1;
  }
  if (submenu == 0 && menu >= 2 && GVSmenuFlag) {
    menu = 0;
    submenu = 1;
    pos = 1;
  }
  if (submenu == 1 && menu == 2) {
    menu = 1;
    pos = 0;
    economyMenu = true;
    // currentTime = millis();
  }
  if (!GVSmenuFlag) {
    if (pos == 3 && menu == 2 && thermmenuflag == LOW && LCDflag == LOW && deltathermflag == LOW) {
      prepos = pos;
      predeltatherm = deltatherm;
      deltathermflag = HIGH;
    }

    if (deltathermflag == HIGH && LCDflag == LOW && thermmenuflag == LOW) {
      u8g.drawFrame(78, 6, 49, 28);
      u8g.drawFrame(77, 5, 51, 30);
      u8g.setFont(u8g_font_gdb20n);
      u8g.setPrintPos(84, 30);
      u8g.print(setTimedTherm - deltatherm, 0);
      if (menu == 2) {
        deltatherm = predeltatherm - (pos - prepos);
      }
      if (deltatherm < 1) {
        predeltatherm = 1;
        pos = 1;
        prepos = pos;
      }

    }

    if (pos == 2 && menu == 2 && thermmenuflag == LOW && LCDflag == LOW && deltathermflag == LOW) {
      prepos = pos;
      presettherm = settherm;
      thermmenuflag = HIGH;
    }

    if (thermmenuflag == HIGH && LCDflag == LOW && deltathermflag == LOW) {
      u8g.drawFrame(78, 6, 49, 28);
      u8g.drawFrame(77, 5, 51, 30);
      u8g.setFont(u8g_font_gdb20n);
      u8g.setPrintPos(84, 30);
      u8g.print(setTimedTherm);
      if (menu == 2) {
        settherm = presettherm + pos - prepos;
        if (daytimeOn) {
          setTimedTherm = settherm - economyAmount;
        }
        else {
          setTimedTherm = settherm;
        }
      }
      if (settherm >= 100 || settherm < 1) {
        prepos = pos;
      }
    }
  }

  else {
    if (pos == 3 && menu == 2 && thermmenuflag == LOW && LCDflag == LOW && deltathermflag == LOW) {
      prepos = pos;
      GVSpredeltatherm = GVSdeltatherm;
      deltathermflag = HIGH;
    }

    if (deltathermflag == HIGH && LCDflag == LOW && thermmenuflag == LOW) {
      u8g.drawFrame(78, 6, 49, 28);
      u8g.drawFrame(77, 5, 51, 30);
      u8g.setFont(u8g_font_gdb20n);
      u8g.setPrintPos(84, 30);
      u8g.print(GVSsettherm - GVSdeltatherm, 0);
      if (menu == 2) {
        GVSdeltatherm = GVSpredeltatherm - (pos - prepos);
      }
      if (deltatherm < 1) {
        GVSpredeltatherm = 1;
        pos = 1;
        prepos = pos;
      }

    }

    if (pos == 2 && menu == 2 && thermmenuflag == LOW && LCDflag == LOW && deltathermflag == LOW) {
      prepos = pos;
      GVSpresettherm = GVSsettherm;
      thermmenuflag = HIGH;
    }

    if (thermmenuflag == HIGH && LCDflag == LOW && deltathermflag == LOW) {
      u8g.drawFrame(78, 6, 49, 28);
      u8g.drawFrame(77, 5, 51, 30);
      u8g.setFont(u8g_font_gdb20n);
      u8g.setPrintPos(84, 30);
      u8g.print(GVSsettherm);
      if (menu == 2) {
        GVSsettherm = GVSpresettherm + pos - prepos;
      }
      if (GVSsettherm >= up_limit || GVSsettherm < 1) {
        prepos = pos;
      }
    }
  }

  if (menu != 2) {
    thermmenuflag = LOW;
    deltathermflag = LOW;
    LCDflag = LOW;
  }

  if (pos == 4 && menu == 2 && LCDflag == LOW && thermmenuflag == LOW && deltathermflag == LOW) {
    preposLED = pos;
    preLCDled = LCDled;
    LCDflag = HIGH;
  }

  if (LCDflag == HIGH && thermmenuflag == LOW && deltathermflag == LOW) {
    u8g.drawBox (120, submenu * 12 , 5, 11);
    LCDled = preLCDled + ((pos - preposLED) * 2.55);
    if (LCDled != preLCDled) {
      analogWrite(LCDledPin, LCDled);
    }
    if (LCDled >= 255 || LCDled < 0) {
      preposLED = pos;
    }
  }
}


void workdraw(void) {
  if (pauseON) {
    u8g.setFont(u8g_font_7x14Br);
    u8g.setPrintPos(77, 40);
    u8g.print("PAUSE");
  }
  else {
    if (error_code > 0) {
      u8g.setFont(u8g_font_4x6r);
      u8g.setPrintPos(76, 36);
      u8g.print("Error");
      u8g.setPrintPos(110, 36);
      u8g.print(error_code);
    }
    else {
      u8g.setFont(u8g_font_7x14Br);
      u8g.setPrintPos(77, 40);
      u8g.print(r_temp41);
    }
  }
  if (!flag_work && trel1) {
    u8g.setFont(u8g_font_4x6r);
    u8g.setPrintPos(79, 9);
    u8g.print("START");
    u8g.setFont(u8g_font_7x14Br);
    u8g.setPrintPos(101, 12);
    u8g.print((total_delay / 60) % 60);
    u8g.setPrintPos(108, 12);
    u8g.print(":");
    u8g.setPrintPos(113, 12);
    u8g.print(total_delay % 60);
  }
  else if (tstop) {
    u8g.setFont(u8g_font_4x6r);
    u8g.setPrintPos(85, 9);
    u8g.print("STOP");
    u8g.setFont(u8g_font_7x14Br);
  }
  else {
    u8g.setFont(u8g_font_7x14Br);
    u8g.setPrintPos(77, 12);
    u8g.print(worktime / 60 / 60);
    u8g.setPrintPos(90, 12);
    u8g.print(":");
    u8g.setPrintPos(95, 12);
    u8g.print((worktime / 60) % 60);
    u8g.setPrintPos(108, 12);
    u8g.print(":");
    u8g.setPrintPos(113, 12);
    u8g.print(worktime % 60);
  }
  u8g.drawFrame(74, 0, 54, 14);
  u8g.drawFrame(74, 15, 54, 14);
  u8g.drawFrame(0, 0, 34, 38);
  u8g.drawFrame(35, 0, 38, 38);

  u8g.setFont(u8g_font_7x14Br);
  u8g.setPrintPos(77, 27);
  u8g.print(Hour);
  u8g.setPrintPos(90, 27);
  u8g.print(":");
  u8g.setPrintPos(95, 27);
  u8g.print(Minute);
  u8g.setPrintPos(108, 27);
  u8g.print(":");
  u8g.setPrintPos(113, 27);
  u8g.print(Second);

  u8g.setPrintPos(4, 12);
  u8g.print("Set:");
  u8g.setPrintPos(38, 12);
  u8g.print("Real:");
  u8g.setFont(u8g_font_gdb20n);
  u8g.setPrintPos(1, 36);
  u8g.print(setTimedTherm);
  u8g.setPrintPos(38, 36);
  // u8g.print(forv_water);
  u8g.print(r_temp11, 0);
  u8g.setFont(u8g_font_7x14Br);
  u8g.setPrintPos(86, 49);
  // u8g.print(rew_water);


  u8g.drawFrame(0, 39, 56, 25);

  u8g.setFont(u8g_font_5x7r);
  u8g.setPrintPos(2, 49);
  u8g.print("In");
  u8g.setPrintPos(2, 60);
  u8g.print("Out");

  u8g.setPrintPos(60, 47);
  u8g.print("GVS Set");

  if (frel6) {
    u8g.setPrintPos(113, 47);
    u8g.print("GVS");
  }
  else {
    u8g.setPrintPos(113, 47);
    u8g.print("DOM");
  }


  u8g.setPrintPos(98, 47);
  u8g.print(GVSsettherm);

  u8g.setPrintPos(60, 55);
  u8g.print("GVS Real");

  u8g.setPrintPos(102, 55);
  u8g.print(GVSrealtherm);


  u8g.setPrintPos(60, 63);
  if (daytimeOn && economyAmount > 0) {
    u8g.print("Economy ON");
  }
  else {
    u8g.print("Economy OFF");
  }


  u8g.setPrintPos(92, 64);
  // u8g.print(temperature);

  u8g.setFont(u8g_font_7x14Br);
  u8g.setPrintPos(20, 51);
  // u8g.print(bunker);
  u8g.print(r_temp21);
  u8g.setPrintPos(20, 62);
  // u8g.print(riser);
  u8g.print(r_temp22);

}

void ecoMenuDraw() {
  u8g.setFont(u8g_font_5x7r);
  u8g.setPrintPos(60, 60);
  if (daytimeOn && economyAmount > 0) {
    u8g.print("Economy ON");
  }
  else {
    u8g.print("Economy OFF");
  }
  u8g.setPrintPos(114, 8);
  u8g.print( (outscreenperiod - (presentTime - outscreentime)) / 1000);
  // u8g.setPrintPos(90, 8);
  // u8g.print(EEPROM.read(33));

  if (ecosaveflag) {
    u8g.setFont(u8g_font_5x7r);
    u8g.setPrintPos(10, 60);
    u8g.print("Saved!");
  }

  if (obnulpos) {
    pos = 0;
    obnulpos = false;
  }

  if (menu == 1) {
    submenu = pos;
    if (submenu >= 4) {
      pos = 0;
    }
    if (submenu < 0) {
      pos = 0;
    }
  }

  if (menu >= 3) {
    economysetflag = false;
    deyendsetflag = false;
    daybegsetflag = false;

    if (EEPROM.read(31) != economyAmount) {
      EEPROM.write(31, economyAmount);
      ecosaveflag = true;
    }
    if (EEPROM.read(32) != daytimeBegin) {
      EEPROM.write(32, daytimeBegin);
      ecosaveflag = true;
    }
    if (EEPROM.read(33) != daytimeEnd) {
      EEPROM.write(33, daytimeEnd);
      ecosaveflag = true;
    }
    menu = 1;
    pos = 0;
  }

  if (submenu == 0 && menu == 2) {
    economyMenu = false;
    economysetflag = false;
    deyendsetflag = false;
    daybegsetflag = false;
    ecosaveflag = false;
    menu = 0;
    submenu = 1;
    pos = 1;
  }
  u8g.drawBox(0, submenu * 12 , 5, 10);
  u8g.drawHLine(0, submenu * 12 + 10 , 120);
  u8g.setFont(u8g_font_7x14Br);
  u8g.setPrintPos(10, 10);
  u8g.print("Back");
  u8g.setPrintPos(10, 22);
  u8g.print("Day begin:");
  u8g.setPrintPos(90, 22);
  u8g.print(daytimeBegin);
  u8g.setPrintPos(10, 34);
  u8g.print("Dey end:");
  u8g.setPrintPos(90, 34);
  u8g.print(daytimeEnd);
  u8g.setPrintPos(10, 46);
  u8g.print("Economy:");
  u8g.setPrintPos(90, 46);
  u8g.print(economyAmount);

  if (menu == 2 && submenu == 1 && !daybegsetflag) {
    pos = 0;
    prepos = pos;
    predaytimeBegin = daytimeBegin;
    daybegsetflag = true;
    deyendsetflag = false;
    economysetflag = false;
  }

  if (menu == 2 && submenu == 2 && !deyendsetflag) {
    pos = 0;
    prepos = pos;
    predaytimeEnd = daytimeEnd;
    deyendsetflag = true;
    economysetflag = false;
    daybegsetflag = false;
  }

  if (menu == 2 && submenu == 3 && !economysetflag) {
    pos = 0;
    prepos = pos;
    preeconomyAmount = economyAmount;
    economysetflag = true;
    deyendsetflag = false;
    daybegsetflag = false;
  }

  if (menu == 2 && daybegsetflag) {
    daytimeBegin = predaytimeBegin + pos - prepos;
    if (daytimeBegin >= daytimeEnd || daytimeBegin < 0) {
      prepos = pos;
    }
  }

  if (menu == 2 && deyendsetflag) {
    daytimeEnd = predaytimeEnd + pos - prepos;
    if (daytimeEnd <= daytimeBegin || daytimeEnd > 24) {
      prepos = pos;
    }
  }

  if (menu == 2 && economysetflag) {
    economyAmount = preeconomyAmount + pos - prepos;
    if (economyAmount < 0 || settherm - economyAmount < 0) {
      prepos = pos;
    }
  }


}

void start_work() {

  if (trel1 && presentTime - total_start_time >= total_start_period) {
    digitalWrite (rel1, LOW); // насос скважины
    if (GVSrealtherm <= GVSsettherm - GVSdeltatherm) {
      digitalWrite (rel6, LOW); // ГВС. насос
      frel6 = true;
      digitalWrite (rel5, HIGH); // отопление. насос
      frel5 = false;
    }
    else {
      digitalWrite (rel6, HIGH); // ГВС. насос
      frel6 = false;
      digitalWrite (rel5, LOW); // отопление. насос
      frel5 = true;
    }
    timer_1_rel = presentTime;
    frel1 = true;
    trel1 = false;
    trel2 = true;
    flag_work = true;
  }

  if (presentTime - timer_1_rel >= timer_1_delay && trel2) {
    digitalWrite (rel2, LOW); // соленоид
    timer_2_rel = presentTime;
    frel2 = true;
    trel2 = false;
    trel3 = true;
  }

  if (presentTime - timer_2_rel >= timer_2_delay && trel3) {
    digitalWrite (rel3, LOW); // компрессор
    timer_3_rel = presentTime;
    frel3 = true;
    trel3 = false;
    trel4 = true;
  }

  if (presentTime - timer_3_rel >= timer_3_delay && trel4) {
    digitalWrite (rel4, LOW); // цирк. насос
    frel4 = true;
    //  digitalWrite (rel7, HIGH); // цирк. насос
    //  frel7 = true;
    timer_4_rel = presentTime;
    trel4 = false;
    trel5 = true;
    error_code = 0; // обнуляем ошибки
  }
}


void stop_work() {
  if (stoptrel3) {
    flag_work = false;
    digitalWrite (rel3, HIGH); // компрессор
    timer_2_rel = presentTime;
    frel3 = false;
    stoptrel3 = false;
    stoptrel2 = true;
  }

  if (presentTime - timer_2_rel >= timer_2_delay && stoptrel2) {
    digitalWrite (rel2, HIGH); // соленоид
    total_start_time = presentTime;
    timer_1_rel = presentTime;
    frel2 = false;
    stoptrel2 = false;
    stoptrel1 = true;
  }

  if (presentTime - timer_1_rel >= timer_1_delay && stoptrel1) {
    digitalWrite (rel1, HIGH); // насос скважины
    timer_4_rel = presentTime;
    frel1 = false;
    stoptrel1 = false;
    stoptrel4 = true;
  }

  if (presentTime - timer_4_rel >= timer_4_delay && stoptrel4) {
    if (pauseON) {
      digitalWrite (rel4, HIGH); // цирк. насос
      frel4 = false;
      digitalWrite (rel6, HIGH); // ГВС. насос
      frel6 = false;
      digitalWrite (rel5, HIGH); // отопление. насос
      frel5 = false;
      digitalWrite (rel7, HIGH); // насос подмеса
      frel7 = false;
    }
    //  digitalWrite (rel6, HIGH); // ГВС. насос
    //  frel6 = false;
    //  digitalWrite (rel5, HIGH); // отопление. насос
    //  frel5 = false;
    stoptrel4 = false;
    stoptrel5 = true;
    tstop = false; // обязательно в конце крайней операции!
  }
}


void eepromread () {
  if (EEPROM.read(1) >= 100 || EEPROM.read(1) <= 0) {
    settherm = 30;
  }
  else {
    settherm = EEPROM.read(1);
  }
  if (EEPROM.read(0) >= 10) {
    LCDled = EEPROM.read(0);
  }
  else {
    LCDled = 80;
  }
  if (EEPROM.read(2) >= 1 && EEPROM.read(2) < 12) {
    deltatherm = EEPROM.read(2);
  }
  else {
    deltatherm = 3;
  }
  if (EEPROM.read(3) >= 1 && EEPROM.read(3) < 10) {
    targetnumber = EEPROM.read(3);
  }
  else {
    targetnumber = 1;
  }
  if (EEPROM.read(11) >= 1 && EEPROM.read(11) < 100) {
    st1 = EEPROM.read(11);
  }
  else {
    st1 = 30;
  }
  if (EEPROM.read(12) >= 1 && EEPROM.read(12) < 100) {
    st2 = EEPROM.read(12);
  }
  else {
    st2 = 30;
  }
  if (EEPROM.read(13) >= 1 && EEPROM.read(13) < 100) {
    st3 = EEPROM.read(13);
    GVSsettherm = st3;
  }
  else {
    st3 = 30;
    GVSsettherm = st3;
  }
  if (EEPROM.read(14) >= 1 && EEPROM.read(14) < 100) {
    st4 = EEPROM.read(14);
  }
  else {
    st4 = 30;
  }
  if (EEPROM.read(15) >= 1 && EEPROM.read(15) < 100) {
    st5 = EEPROM.read(15);
  }
  else {
    st5 = 30;
  }
  if (EEPROM.read(21) >= 1 && EEPROM.read(21) < 50) {
    delta1 = EEPROM.read(21);
  }
  else {
    delta1 = 3;
  }
  if (EEPROM.read(22) >= 1 && EEPROM.read(22) < 50) {
    delta2 = EEPROM.read(22);
  }
  else {
    delta2 = 3;
  }
  if (EEPROM.read(23) >= 1 && EEPROM.read(23) < 50) {
    delta3 = EEPROM.read(23);
    GVSdeltatherm = delta3;
  }
  else {
    delta3 = 3;
    GVSdeltatherm = delta3;
  }
  if (EEPROM.read(24) >= 1 && EEPROM.read(24) < 50) {
    delta4 = EEPROM.read(24);
  }
  else {
    delta4 = 3;
  }
  if (EEPROM.read(25) >= 1 && EEPROM.read(25) < 50) {
    delta5 = EEPROM.read(25);
  }
  else {
    delta5 = 3;
  }
  ////////
  if (EEPROM.read(31) >= 50) {
    economyAmount = 3;
  }
  else {
    economyAmount = EEPROM.read(31);
  }
  if (EEPROM.read(32) >= 0 && EEPROM.read(32) < 24) {
    daytimeBegin = EEPROM.read(32);
  }
  else {
    daytimeBegin = 7;
  }
  if (EEPROM.read(33) >= 0 && EEPROM.read(33) < 24) {
    daytimeEnd = EEPROM.read(33);
  }
  else {
    daytimeEnd = 22;
  }
  //////////////////
  if (EEPROM.read(111) >= 250 || EEPROM.read(111) <= 0) {
    adj_temp11 = 0;
  }
  else {
    adj_temp11 = (EEPROM.read(111) - 100) * 0.2;
  }
  if (EEPROM.read(112) >= 250 || EEPROM.read(112) <= 0) {
    adj_temp12 = 0;
  }
  else {
    adj_temp12 = (EEPROM.read(112) - 100) * 0.2;
  }

  if (EEPROM.read(121) >= 250 || EEPROM.read(121) <= 0) {
    adj_temp21 = 0;
  }
  else {
    adj_temp21 = (EEPROM.read(121) - 100) * 0.2;
  }
  if (EEPROM.read(122) >= 250 || EEPROM.read(122) <= 0) {
    adj_temp22 = 0;
  }
  else {
    adj_temp22 = (EEPROM.read(122) - 100) * 0.2;
  }

  if (EEPROM.read(131) >= 250 || EEPROM.read(131) <= 0) {
    adj_temp31 = 0;
  }
  else {
    adj_temp31 = (EEPROM.read(131) - 100) * 0.2;
  }
  if (EEPROM.read(132) >= 250 || EEPROM.read(132) <= 0) {
    adj_temp32 = 0;
  }
  else {
    adj_temp32 = (EEPROM.read(132) - 100) * 0.2;
  }

  if (EEPROM.read(141) >= 250 || EEPROM.read(141) <= 0) {
    adj_temp41 = 0;
  }
  else {
    adj_temp41 = (EEPROM.read(141) - 100) * 0.2;
  }
  if (EEPROM.read(142) >= 250 || EEPROM.read(142) <= 0) {
    adj_temp42 = 0;
  }
  else {
    adj_temp42 = (EEPROM.read(142) - 100) * 0.2;
  }

  if (EEPROM.read(151) >= 250 || EEPROM.read(151) <= 0) {
    adj_temp51 = 0;
  }
  else {
    adj_temp51 = (EEPROM.read(151) - 100) * 0.2;
  }
  if (EEPROM.read(152) >= 250 || EEPROM.read(152) <= 0) {
    adj_temp52 = 0;
  }
  else {
    adj_temp52 = (EEPROM.read(152) - 100) * 0.2;
  }
}

void GVS_HOME_relations() {

  if (frel1) {
    if (GVSrealtherm <= GVSsettherm - GVSdeltatherm && td31) {
      digitalWrite (rel6, LOW); // ГВС. насос
      frel6 = true;
      digitalWrite (rel5, HIGH); // отопление. насос
      frel5 = false;
    }
  }

  if ((GVSrealtherm >= GVSsettherm || !td31) || !frel1 && error_code != 61) {
    digitalWrite (rel6, HIGH); // ГВС. насос
    frel6 = false;
    digitalWrite (rel5, LOW); // отопление. насос
    frel5 = true;
  }

  if (frel1 && setTimedTherm <= realtherm && GVSrealtherm < GVSsettherm && td31) {
    digitalWrite (rel6, LOW); // ГВС. насос
    frel6 = true;
    digitalWrite (rel5, HIGH); // отопление. насос
    frel5 = false;
  }

  if (r_temp41 >= up_limit - podmes_delta) {
    digitalWrite (rel7, LOW); // насос подмеса
    frel7 = true;
  }
  if (r_temp41 <= GVSsettherm + 1 || r_temp41 <= up_limit - low_podmes_delta) {
    digitalWrite (rel7, HIGH); // насос подмеса
    frel7 = false;
  } if (frel1) {
    if (GVSrealtherm <= GVSsettherm - GVSdeltatherm && td31) {
      digitalWrite (rel6, LOW); // ГВС. насос
      frel6 = true;
      digitalWrite (rel5, HIGH); // отопление. насос
      frel5 = false;
    }
  }

  if ((GVSrealtherm >= GVSsettherm || !td31) || !frel1 && error_code != 61) {
    digitalWrite (rel6, HIGH); // ГВС. насос
    frel6 = false;
    digitalWrite (rel5, LOW); // отопление. насос
    frel5 = true;
  }

  if (frel1 && setTimedTherm <= realtherm && GVSrealtherm < GVSsettherm && td31) {
    digitalWrite (rel6, LOW); // ГВС. насос
    frel6 = true;
    digitalWrite (rel5, HIGH); // отопление. насос
    frel5 = false;
  }

  if (r_temp41 >= up_limit - podmes_delta) {
    digitalWrite (rel7, LOW); // насос подмеса
    frel7 = true;
  }
  if (r_temp41 <= GVSsettherm + 1 || r_temp41 <= up_limit - low_podmes_delta) {
    digitalWrite (rel7, HIGH); // насос подмеса
    frel7 = false;
  }
}

void Errors_List() {

  if (digitalRead (RD_pin) == HIGH) {
    must_to_stop = true;
    error_code = 911; // номер ошибки
  }

  if (!td11 && presentTime - error_1_time > errordelay && setTimedTherm > 3) { // даем время на поиск датчика прежде выдачи ошибки
    error_code = 1; // номер ошибки
    must_to_stop = true;
  }

  if (!td21 && presentTime - error_2_time > errordelay) { // даем время на поиск датчика прежде выдачи ошибки
    error_code = 2; // номер ошибки
    // must_to_stop = true;
  }

  if (!td31 && presentTime - error_3_time > errordelay && GVSsettherm > 3) { // даем время на поиск датчика прежде выдачи ошибки
    error_code = 3; // номер ошибки
    must_to_stop = true;
  }

  if (!td41 && presentTime - error_4_time > errordelay) { // даем время на поиск датчика прежде выдачи ошибки
    error_code = 4; // номер ошибки
    must_to_stop = true;
  }

  if (!td51 && presentTime - error_5_time > errordelay) { // даем время на поиск датчика прежде выдачи ошибки
    // error_code = 5; // номер ошибки
    //must_to_stop = true;
  }

  if (r_temp41 >= up_limit) {
    error_code = 61; // номер ошибки
    must_to_stop = true;
  }

  if (error_code != 0 && error_code != 61 && error_code != 31) {
    tone(speakerPin, 2000, 200);
    delay (200);
  }

}

void setup() {
  pinMode(LCDledPin, OUTPUT);

  pinMode(speakerPin, OUTPUT);
  tone(speakerPin, 1200, 100);

  eepromread ();

  pinMode( tempPin1, INPUT );
  pinMode( tempPin2, INPUT );
  pinMode( tempPin3, INPUT );
  pinMode( tempPin4, INPUT );
  pinMode( tempPin5, INPUT );
  pinMode( tempPin6, INPUT );
  pinMode( tempPin7, INPUT );
  pinMode( tempPin8, INPUT );
  pinMode( tempPin9, INPUT );


  pinMode(RD_pin, INPUT_PULLUP);

  pinMode(RDsupply_pin, OUTPUT);
  digitalWrite (RDsupply_pin, LOW);

  pinMode(rel1, OUTPUT);
  digitalWrite (rel1, HIGH);
  frel1 = false;
  pinMode(rel2, OUTPUT);
  digitalWrite (rel2, HIGH);
  frel2 = false;
  pinMode(rel3, OUTPUT);
  digitalWrite (rel3, HIGH);
  frel3 = false;
  pinMode(rel4, OUTPUT);
  digitalWrite (rel4, LOW);// включаем цирк.насос
  frel4 = true;
  pinMode(rel5, OUTPUT);
  digitalWrite (rel5, LOW);// включаем трехходовик по отоплению
  frel5 = true;
  pinMode(rel6, OUTPUT);
  digitalWrite (rel6, HIGH);
  frel6 = false;
  pinMode(rel7, OUTPUT);
  digitalWrite (rel7, HIGH);
  frel7 = false;
  pinMode(rel8, OUTPUT);
  digitalWrite (rel8, HIGH);
  frel8 = false;

  analogWrite(LCDledPin, LCDled);

  Serial.begin(9600);
  Serial.println("Serial is Ok!");


  currentTime = millis();

  attachInterrupt(0, Button, CHANGE); //прерывание по изменению пина №2
  Timer1.initialize(250); // инициализация таймера 1, период 250 мкс
  Timer1.attachInterrupt(timerInterrupt, 250); // задаем обработчик прерываний

  // stepper.setSpeed(13); //установка скорости вращения ротора


  total_start_time = millis();
  pos = 0;
  flag_work = true;
  must_to_stop = false;
  economyMenu = false;
  economysetflag = false;
  deyendsetflag = false;
  daybegsetflag = false;
  ecosaveflag = false;
}

void loop() {

  presentTime = millis();
  worktime = (presentTime - currentTime) / 1000; // таймер времени работ

  /////////////// Обработка длинного нажатия
  if (!buttflag) {
    START_STOPtime = presentTime;
  }

  if (buttflag && presentTime - START_STOPtime >= buttonSTART_STOPperiod) {
    longClick = true;
    menu = 0;
    pos = 1;
    START_STOPtime = presentTime;
  }

  if (longClick && !pauseON && sensScreenOn && !enginMenuOn) {
    enginMenuOn = true;
    pos = 0;
    tone(speakerPin, 1800, 500);
  }

  if (longClick && !pauseON && !sensScreenOn && !enginMenuOn) {
    pauseON = true;
    longClick = false;
    must_to_stop = true;
    frel3 = true;
    LED_time = presentTime - LED_delay;
    tone(speakerPin, 400, 500);
  }

  if (longClick && pauseON) {
    pauseON = false;
    longClick = false;
    menu = 0;
    pos = 1;
    digitalWrite (rel4, LOW);// включаем цирк.насос
    frel4 = true;
    tone(speakerPin, 1000, 500);
  }
  ////////////////// Работа с часами

  if (Hour >= daytimeBegin && Hour < daytimeEnd) {
    daytimeOn = true;
  }
  else {
    daytimeOn = false;
  }

  if (daytimeOn) {
    setTimedTherm = settherm - economyAmount;
  }
  else {
    setTimedTherm = settherm;
  }

  if (RTC.read(tm)) {
    Hour = tm.Hour;
    Minute = tm.Minute;
    Second = tm.Second;
    Day = tm.Day;
    Month = tm.Month;
    Year = tmYearToCalendar(tm.Year);
  }
  else {
    if (RTC.chipPresent()) {
      Serial.println("The DS1307 is stopped.  Please run the SetTime");
      Serial.println("example to initialize the time and begin running.");
      Serial.println();
      error_code = 31; // номер ошибки
    } else {
      // Serial.println("DS1307 read error!  Please check the circuitry.");
      // Serial.println();
      // error_code = 32; // номер ошибки
    }
  }
  /////////////////// Отсчет задержки включения

  if (!flag_work) {
    total_delay = (total_start_period - (presentTime - total_start_time)) / 1000;
  }

  ///////////////// Получение реальных температур для управления

  if (presentTime - sensorstime >= sensorsperiod && (menu == 0 || enginMenuOn)) {
    readsensors();
    sensorstime = presentTime;
  }
  realtherm = r_temp11;
  GVSrealtherm = r_temp31;

  /////////////////  Условия старта насоса при падении температур

  if ((setTimedTherm - realtherm >= deltatherm && td11) || (GVSsettherm - GVSrealtherm  >= GVSdeltatherm && td31) && !tstop && !pauseON) {
    if (flag_work && !frel1) {
      flag_work = false;
    }
    if (!frel1) {
      trel1 = true;
      currentTime = millis(); // засекаем время работы насоса
    }
    start_work();
  }
  else {
    flag_work = true;
  }

  //////////////// Условия остановки насоса после набора температур

  if ((setTimedTherm <= realtherm || !td11) && (GVSsettherm <= GVSrealtherm || !td31) && !tstop) {
    must_to_stop = true;
  }
  ///////////////////  Если не на паузе, то переключаем ГВС/отопление о отслеживаем ошибки

  if (!pauseON) {
    GVS_HOME_relations();
    Errors_List();
  }

  //  Serial.print (" menu  ");
  //  Serial.print (menu);
  //  Serial.print (" sens  ");
  //  Serial.print (sensScreenOn);
  //  Serial.print (" engin  ");
  //  Serial.print (enginMenuOn);
  //  Serial.print ("  pos  ");
  //  Serial.println (pos);
  //  Serial.print ("  ");
  //  Serial.print (deltatherm);
  //  Serial.print ("  ");
  //  Serial.println (settherm - realtherm);

  /////////////////  Обработка команды остановки насоса
  if (must_to_stop || tstop) {
    if (!flag_work) {
      flag_work = true;
    }
    if (frel3 || trel3) {
      stoptrel3 = true;
      tstop = true;
    }
    stop_work();
    if (frel3) {
      currentTime = millis(); // засекаем время простоя насоса
    }
    must_to_stop = false;
  }



  // листаем экраны
  if (economyMenu) {
    u8g.firstPage();
    do {
      ecoMenuDraw();
    } while ( u8g.nextPage() );
  }

  if (pos <= 1 && menu == 0 && !economyMenu && !enginMenuOn) {
    sensScreenOn = false;
    u8g.firstPage();
    do {
      workdraw();
    } while ( u8g.nextPage() );
  }

  if (pos == 2 && menu == 0 && !economyMenu && !enginMenuOn) {
    sensScreenOn = true;
    u8g.firstPage();
    do {
      sensorsdraw();
    } while ( u8g.nextPage() );
  }


  if (pos == 3 && menu == 0 && !economyMenu && !enginMenuOn) {
    sensScreenOn = false;
    u8g.firstPage();
    do {
      releydraw();
    } while ( u8g.nextPage() );
  }

  if (pos == 4 && menu == 0 && !economyMenu && !enginMenuOn) {
    sensScreenOn = false;
    u8g.firstPage();
    do {
      eepromdraw();
    } while ( u8g.nextPage() );
  }

  if (menu >= 1 && !economyMenu && !enginMenuOn) {
    error_code = 0;
    u8g.firstPage();
    do {
      menudraw();
    } while ( u8g.nextPage() );
  }
  else if (!economyMenu && !enginMenuOn) {
    obnulpos = true;
    GVSmenuFlag = false;
  }

  if (enginMenuOn) {
    // enginMenuOn = true;
    u8g.firstPage();
    do {
      igeneer_menudraw();
    } while ( u8g.nextPage() );
  }


  ///////////////////// Управляем подсветкой экрана и возвратом на рабочий экран по таймеру


  if (pos == 1 || (outscreenOn && pos != 1)) {
    outscreentime = presentTime;
    outscreenOn = false;
  }

  if (presentTime - outscreentime >= outscreenperiod && !outscreenOn && pos != 1 && !enginMenuOn) {
    pos = 1;
    menu = 0;
    economyMenu = false;
    tone(speakerPin, 1100, 250);
    tone(speakerPin, 1000, 250);
    outscreenOn = true;
  }

  if (pos != preposLED_on || menu != 0) {
    preposLED_on = pos;
    if (!pauseON) {
      LED_time = presentTime;
    }
    if (LED_on) {
      analogWrite(LCDledPin, LCDled);
      LED_on = false;
    }
  }

  if (presentTime - LED_time >= LED_delay && !enginMenuOn) {
    if (!pauseON) {
      analogWrite(LCDledPin, 5);
    }
    else {
      analogWrite(LCDledPin, 0);
    }
    menu = 0;
    economyMenu = false;
    if (!LED_on) {
      LED_on = true;
    }
  }

}
