#include <RTClib.h>
#include <LiquidCrystal_I2C.h>
#include <EEPROM.h>
#include <DFPlayer_Mini_Mp3.h>
#include <math.h>

#define NUM_OPCIONES_MENU 5

#define PANTALLA_INICIAL 0
#define PANTALLA_MENU 1
#define PANTALLA_CONFIG_FECHA_HORA 2
#define PANTALLA_CONFIG_MEGAFONIA 3
#define PANTALLA_CONFIG_PORTAL 4
#define PANTALLA_CONFIG_DIAS_SEMANA 5

#define MENU_OPTION_CONFIG_FECHA_HORA 0
#define MENU_OPTION_CONFIG_MEGAFONIA 1
#define MENU_OPTION_CONFIG_PORTAL 2
#define MENU_OPTION_CONFIG_DIAS_SEMANA 3
#define MENU_OPTION_ATRAS 4

#define CONFIG_FECHA_DIA 0
#define CONFIG_FECHA_MES 1
#define CONFIG_FECHA_ANO 2
#define CONFIG_FECHA_HORA 3
#define CONFIG_FECHA_MINUTO 4

#define CONFIG_MEGAFONIA_HORA 0
#define CONFIG_MEGAFONIA_MINUTO 1

#define CONFIG_PORTAL_HORA_ABRIR 0
#define CONFIG_PORTAL_MINUTO_ABRIR 1
#define CONFIG_PORTAL_HORA_CERRAR 2
#define CONFIG_PORTAL_MINUTO_CERRAR 3

#define PIN_SALIDA_MEGAFONIA 22
#define PIN_SALIDA_PORTAL 24
#define PIN_PILOTO_HORARIO 26

#define PIN_BOTON_ARRIBA 23
#define PIN_BOTON_ABAIXO 25
#define PIN_BOTON_OK 27
#define PIN_ESTADO_SISTEMA 29

LiquidCrystal_I2C lcd(0x27, 20, 4);
DateTime fechaActual;
DateTime megafoniaHora;
DateTime portalAbrirHora;
DateTime portalCerrarHora;

RTC_DS3231 rtc;

bool mostrarDosPuntos;
char configDiasSemanaArray[7] = {' ', ' ', ' ', ' ', ' ', ' ', ' ' };
byte diasSemana = 0;
bool estaPortaAberta = false;
bool estaMegafoniaFuncionando = false;

unsigned long previousTimeDosPuntos = 0;
unsigned long previousTimeOperation = 0;
bool tempTime = false;
bool cambiouseHorario = false;

/* EEPROM MEGA bytes
    -- byte 0: Dias de semana de activacion (D-S)
    -- byte 1: hora megafonia
    -- byte 2: minuto megafonia
    -- byte 3: hora apertura portal
    -- byte 4: minuto apertura portal
    -- byte 5: hora peche portal
    -- byte 6: minuto peche portal
    -- byte 7: estado cambio horario
*/

void setup() {
  lcd.begin();
  lcd.clear();

  Serial.begin(9600);
  if (!rtc.begin()) {
    Serial.println(F("Couldn't find RTC"));
    while(!rtc.begin());
  }

  if (rtc.lostPower()) {
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }
  
  mostrarDosPuntos = false;
  megafoniaHora = DateTime(0, 0, 0, EEPROM.read(1), EEPROM.read(2), 0);
  portalAbrirHora = DateTime(0, 0, 0, EEPROM.read(3), EEPROM.read(4), 0);
  portalCerrarHora = DateTime(0, 0, 0, EEPROM.read(5), EEPROM.read(6), 0);
  diasSemana = EEPROM.read(0);

  for(char i = 0; i < 7; i++){
    if((diasSemana >> i) & 1){
      configDiasSemanaArray[i] = '*';
    }
  }

  cambiouseHorario = (bool) EEPROM.read(7);

  Serial2.begin(9600);
  mp3_set_serial(Serial2);
  mp3_set_volume(30);
  mp3_single_loop(false);
  mp3_set_EQ(2);
  
  pinMode(PIN_SALIDA_MEGAFONIA, OUTPUT);
  pinMode(PIN_SALIDA_PORTAL, OUTPUT);
  pinMode(PIN_PILOTO_HORARIO, OUTPUT);
  digitalWrite(PIN_SALIDA_MEGAFONIA, HIGH);
  digitalWrite(PIN_SALIDA_PORTAL, HIGH);
  digitalWrite(PIN_PILOTO_HORARIO, HIGH);
  
  pinMode(PIN_BOTON_ARRIBA, INPUT_PULLUP);
  pinMode(PIN_BOTON_ABAIXO, INPUT_PULLUP);
  pinMode(PIN_BOTON_OK, INPUT_PULLUP);
  pinMode(PIN_ESTADO_SISTEMA, INPUT_PULLUP);
}

char pantalla = PANTALLA_INICIAL;
char selectOption = 0;
char selectEstadoSistema = 0;

void loop() {
  lcd.noCursor();
  
  switch(pantalla){
    case PANTALLA_INICIAL:
      fechaActual = rtc.now();
      pantallaIniciar(fechaActual);
      doOperation(fechaActual, portalAbrirHora, portalCerrarHora, megafoniaHora);
      comprobarCambioHorario(fechaActual);
      break;
    case PANTALLA_MENU:
      pantallaMenu();
      fechaActual = rtc.now();
      doOperation(fechaActual, portalAbrirHora, portalCerrarHora, megafoniaHora);
      comprobarCambioHorario(fechaActual);
      break;
    case PANTALLA_CONFIG_FECHA_HORA:
      pantallaConfigFechaHora(fechaActual);
      break;
    case PANTALLA_CONFIG_MEGAFONIA:
      pantallaConfigMegafonia(megafoniaHora);
      break;
    case PANTALLA_CONFIG_PORTAL:
      pantallaConfigPortal(portalAbrirHora, portalCerrarHora);
      break;
    case PANTALLA_CONFIG_DIAS_SEMANA:
      pantallaConfigDiasSemana();
      break;      
  }
  leerSerial();
  leerInput();
  
  delay(20);
}

char pantallaAnterior = PANTALLA_INICIAL;
char configFechaPos = CONFIG_FECHA_DIA;
char configMegafoniaPos = CONFIG_MEGAFONIA_HORA;
char configPortalPos = CONFIG_PORTAL_HORA_ABRIR;
char configDiasSemanaPos = 0;

DateTime modificarFecha(DateTime fecha, char op, char tipo){
  //Restar
  if(op == 0){
    switch(tipo){
      case 0:
        if(fecha.day() > 1){
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day() - 1, fecha.hour(), fecha.minute(), fecha.second()); 
        }
        break;
      case 1:
        if(fecha.month() > 1){
          fecha = DateTime(fecha.year(), fecha.month() - 1, fecha.day(), fecha.hour(), fecha.minute(), fecha.second()); 
        }
        break;
      case 2:
        if(fecha.year() > 2018){
          fecha = DateTime(fecha.year() - 1, fecha.month(), fecha.day(), fecha.hour(), fecha.minute(), fecha.second()); 
        }
        break;
      case 3:
        if(fecha.hour() > 0){
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day(), fecha.hour() - 1, fecha.minute(), fecha.second()); 
        }
        else {
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day(), 23, fecha.minute(), fecha.second()); 
        }
        break;
      case 4:
        if(fecha.minute() > 0){
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day(), fecha.hour(), fecha.minute() - 1, fecha.second()); 
        }
        else {
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day(), fecha.hour(), 59, fecha.second()); 
        }
        break;
    }
  }
  else if(op == 1){ //Sumar
    switch(tipo){
      case 0:
        if(fecha.day() < 31){
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day() + 1, fecha.hour(), fecha.minute(), fecha.second()); 
        }
        break;
      case 1:
        if(fecha.month() < 12){
          fecha = DateTime(fecha.year(), fecha.month() + 1, fecha.day(), fecha.hour(), fecha.minute(), fecha.second()); 
        }
        break;
      case 2:
        fecha = DateTime(fecha.year() + 1, fecha.month(), fecha.day(), fecha.hour(), fecha.minute(), fecha.second()); 
        break;
      case 3:
        if(fecha.hour() < 23){
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day(), fecha.hour() + 1, fecha.minute(), fecha.second()); 
        }
        else {
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day(), 0, fecha.minute(), fecha.second()); 
        }
        break;
      case 4:
        if(fecha.minute() < 59){
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day(), fecha.hour(), fecha.minute() + 1, fecha.second()); 
        }
        else {
          fecha = DateTime(fecha.year(), fecha.month(), fecha.day(), fecha.hour(), 0, fecha.second()); 
        }
        break;
    }
  }
  
  return fecha;
}

void opciones(char option){
  pantallaAnterior = pantalla;
  
  if(option=='a' && pantalla != 0) {
    switch(pantalla){
      case PANTALLA_MENU:
        selectOption--;
        if(selectOption < 0){
          selectOption = NUM_OPCIONES_MENU - 1;
        }
        break;
       case PANTALLA_CONFIG_FECHA_HORA:
        switch(configFechaPos){
          case CONFIG_FECHA_DIA:
            fechaActual = modificarFecha(fechaActual, 0, 0);
            break;
          case CONFIG_FECHA_MES:
            fechaActual = modificarFecha(fechaActual, 0, 1);
            break;
          case CONFIG_FECHA_ANO:
            fechaActual = modificarFecha(fechaActual, 0, 2);
            break;
          case CONFIG_FECHA_HORA:
            fechaActual = modificarFecha(fechaActual, 0, 3);
            break;
          case CONFIG_FECHA_MINUTO:
            fechaActual = modificarFecha(fechaActual, 0, 4);
            break;
        }
        break;
      case PANTALLA_CONFIG_MEGAFONIA:
        switch(configMegafoniaPos){
          case CONFIG_MEGAFONIA_HORA:
            megafoniaHora = modificarFecha(megafoniaHora, 0, 3);
          break;
          case CONFIG_MEGAFONIA_MINUTO:
            megafoniaHora = modificarFecha(megafoniaHora, 0, 4);
          break;
        }
      break;
      case PANTALLA_CONFIG_PORTAL:
          switch(configPortalPos){
            case CONFIG_PORTAL_HORA_ABRIR:
              portalAbrirHora = modificarFecha(portalAbrirHora, 0, 3);
              break;
            case CONFIG_PORTAL_MINUTO_ABRIR:
              portalAbrirHora = modificarFecha(portalAbrirHora, 0, 4);
              break;
            case CONFIG_PORTAL_HORA_CERRAR:
              portalCerrarHora = modificarFecha(portalCerrarHora, 0, 3);
              break;
            case CONFIG_PORTAL_MINUTO_CERRAR:
              portalCerrarHora = modificarFecha(portalCerrarHora, 0, 4);
              break;
          }
        break;
        case PANTALLA_CONFIG_DIAS_SEMANA:
          if(configDiasSemanaPos > 0){
            configDiasSemanaPos--;
          }
        break;
    }
  }
  else if(option=='s' && pantalla != 0) {
    switch(pantalla){
      case PANTALLA_MENU:
        selectOption++;
        if(selectOption > NUM_OPCIONES_MENU - 1){
          selectOption = 0;
        }
        break;
      case PANTALLA_CONFIG_FECHA_HORA:
        switch(configFechaPos){
          case CONFIG_FECHA_DIA:
            fechaActual = modificarFecha(fechaActual, 1, 0);
            break;
          case CONFIG_FECHA_MES:
            fechaActual = modificarFecha(fechaActual, 1, 1);
            break;
          case CONFIG_FECHA_ANO:
            fechaActual = modificarFecha(fechaActual, 1, 2);
            break;
          case CONFIG_FECHA_HORA:
            fechaActual = modificarFecha(fechaActual, 1, 3);
            break;
          case CONFIG_FECHA_MINUTO:
            fechaActual = modificarFecha(fechaActual, 1, 4);
            break;
        }
        break;
      case PANTALLA_CONFIG_MEGAFONIA:
        switch(configMegafoniaPos){
          case CONFIG_MEGAFONIA_HORA:
            megafoniaHora = modificarFecha(megafoniaHora, 1, 3);
          break;
          case CONFIG_MEGAFONIA_MINUTO:
            megafoniaHora = modificarFecha(megafoniaHora, 1, 4);
          break;
        }
        break;
      case PANTALLA_CONFIG_PORTAL:
          switch(configPortalPos){
            case CONFIG_PORTAL_HORA_ABRIR:
              portalAbrirHora = modificarFecha(portalAbrirHora, 1, 3);
              break;
            case CONFIG_PORTAL_MINUTO_ABRIR:
              portalAbrirHora = modificarFecha(portalAbrirHora, 1, 4);
              break;
            case CONFIG_PORTAL_HORA_CERRAR:
              portalCerrarHora = modificarFecha(portalCerrarHora, 1, 3);
              break;
            case CONFIG_PORTAL_MINUTO_CERRAR:
              portalCerrarHora = modificarFecha(portalCerrarHora, 1, 4);
              break;
          }
        break;
      case PANTALLA_CONFIG_DIAS_SEMANA:
        if(configDiasSemanaPos < 8){
          configDiasSemanaPos++;
        }
        break;
    }
  }
  else if(option=='d'){
    switch(pantalla){
      case PANTALLA_INICIAL:
        pantalla = PANTALLA_MENU;
        break;
      case PANTALLA_MENU:
        switch(selectOption){
          case MENU_OPTION_CONFIG_FECHA_HORA:
            fechaActual = rtc.now();
            pantalla = PANTALLA_CONFIG_FECHA_HORA;
            break;
          case MENU_OPTION_CONFIG_MEGAFONIA:
            pantalla = PANTALLA_CONFIG_MEGAFONIA;
            break;
          case MENU_OPTION_CONFIG_PORTAL:
            pantalla = PANTALLA_CONFIG_PORTAL;
            break;
          case MENU_OPTION_CONFIG_DIAS_SEMANA:
            pantalla = PANTALLA_CONFIG_DIAS_SEMANA;
            break;
          case MENU_OPTION_ATRAS:
            pantalla = PANTALLA_INICIAL;
            break;
        }
        break;
      case PANTALLA_CONFIG_FECHA_HORA:
        if(configFechaPos == CONFIG_FECHA_MINUTO){
          rtc.adjust(fechaActual);
          pantalla = PANTALLA_INICIAL;
          configFechaPos = CONFIG_FECHA_DIA;
        }
        else {
          configFechaPos++;
        }
        break;
      case PANTALLA_CONFIG_MEGAFONIA:
        if(configMegafoniaPos == CONFIG_MEGAFONIA_MINUTO){
          EEPROM.write(1, megafoniaHora.hour());
          EEPROM.write(2, megafoniaHora.minute());
          pantalla = PANTALLA_INICIAL;
          configMegafoniaPos = CONFIG_MEGAFONIA_HORA;
        }
        else {
          configMegafoniaPos++;
        }
        break;
      case PANTALLA_CONFIG_PORTAL:
        if(configPortalPos == CONFIG_PORTAL_MINUTO_CERRAR){
          EEPROM.write(3, portalAbrirHora.hour());
          EEPROM.write(4, portalAbrirHora.minute());
          EEPROM.write(5, portalCerrarHora.hour());
          EEPROM.write(6, portalCerrarHora.minute());
          pantalla = PANTALLA_INICIAL;
          configPortalPos = CONFIG_PORTAL_HORA_ABRIR;
        }
        else {
          configPortalPos++;
        }
        break;
      case PANTALLA_CONFIG_DIAS_SEMANA:
        if(configDiasSemanaPos < 7){
          if(configDiasSemanaArray[configDiasSemanaPos] == ' '){
            configDiasSemanaArray[configDiasSemanaPos] = '*';
          }
          else {
            configDiasSemanaArray[configDiasSemanaPos] = ' ';
          }
        }
        else {
          diasSemana = 0;
          for(char i = 0; i < 7 ; i++){
            if(configDiasSemanaArray[i] == '*'){
              diasSemana += (byte) (pow(2, i) + 0.5);
            }
          }
          EEPROM.write(0, diasSemana);

          configDiasSemanaPos = 0;
          pantalla =  PANTALLA_INICIAL;
        }
        break;
    }
  }

  if(pantallaAnterior != pantalla){
    lcd.clear();
    selectOption = MENU_OPTION_CONFIG_FECHA_HORA;
  }
}

void leerSerial(){
  if (Serial.available()>0){
    char option=Serial.read();
    opciones(option);
  }
}

void leerInput(){
  if (digitalRead(PIN_BOTON_OK) == LOW) {
    opciones('d');
    while(digitalRead(PIN_BOTON_OK) == LOW);
  }

  if (digitalRead(PIN_BOTON_ARRIBA) == LOW) {
    opciones('a');
    while(digitalRead(PIN_BOTON_ARRIBA) == LOW);
  }

  if (digitalRead(PIN_BOTON_ABAIXO) == LOW) {
    opciones('s');
    while(digitalRead(PIN_BOTON_ABAIXO) == LOW);
  }
}

void pantallaIniciar(DateTime date){
  String blink = ":";
  if(mostrarDosPuntos){
    blink = " ";
  }

  String dia;
  String mes;
  String hora;
  String minutos;
  
  if(date.day() < 10){
    dia = "0" + String(date.day());
  }
  else {
    dia = String(date.day());
  }

  if(date.month() < 10){
    mes = "0" + String(date.month());
  }
  else {
    mes = String(date.month());
  }

  if(date.hour() < 10){
    hora = "0" + String(date.hour());
  }
  else {
    hora = String(date.hour());
  }

  if(date.minute() < 10){
    minutos = "0" + String(date.minute());
  }
  else {
    minutos = String(date.minute());
  }
  
  String fecha = dia + "/" + mes + "/" + String(date.year()) + " - " + hora + blink + minutos;
  lcd.setCursor(0,0);
  lcd.print("Fecha y hora:");
  lcd.setCursor(0,1);
  lcd.print(fecha);

  lcd.setCursor(0, 3);
  String estadoSistemaStr = "Desactivado";
  if(digitalRead(PIN_ESTADO_SISTEMA) == LOW){
    estadoSistemaStr = "Activado   ";
  }
  lcd.print("Estado: " + estadoSistemaStr);

  unsigned long currentTime = millis();
  if(currentTime - previousTimeDosPuntos >= 500){
    previousTimeDosPuntos = currentTime;
    mostrarDosPuntos = !mostrarDosPuntos;
  }
}

char isClearMenuLcd = 0;
void pantallaMenu(){
  String menuOpciones[NUM_OPCIONES_MENU] = {
    "Conf Fecha        ",
    "Conf Megafonia    ",
    "Conf Portal       ",
    "Conf Dias Semana  ",
    "Atras             "
  };


  char start = 0;
  if(selectOption/3.0 > 1.0){
    
    start = selectOption % 3;

    if(isClearMenuLcd == 0){
      isClearMenuLcd = 1;
      lcd.clear();
    }
  }
  else {
    isClearMenuLcd = 0;
  }
  
  char contador = 0;
  for(char i = start; i < NUM_OPCIONES_MENU && contador < 4; i++){
    lcd.setCursor(0, i - start);
    String a = "  ";
    if(i == selectOption){
      a = "> ";
    }
    lcd.print(a + menuOpciones[i]);

    contador++;
  }
}

void pantallaConfigFechaHora(DateTime fecha){
  String dia = String(fecha.day());
  String mes = String(fecha.month());
  String hora = String(fecha.hour());
  String minuto = String(fecha.minute());

  if(fecha.day() < 10){
    dia = "0" + String(fecha.day());
  }
  
  if(fecha.month() < 10){
    mes = "0" + String(fecha.month());
  }

  if(fecha.hour() < 10){
    hora = "0" + String(fecha.hour());
  }

  if(fechaActual.minute() < 10){
    minuto = "0" + String(fecha.minute());
  }

  lcd.setCursor(0,0); 
  lcd.print("Conf Fecha:");
  lcd.setCursor(0,1);
  lcd.print(dia + "/" + mes + "/" + String(fecha.year()) + " - " + hora + ":" + minuto);

  switch(configFechaPos){
    case CONFIG_FECHA_DIA:
      lcd.setCursor(0,1);
      break;
    case CONFIG_FECHA_MES:
      lcd.setCursor(3,1);
      break;
    case CONFIG_FECHA_ANO:
      lcd.setCursor(6,1);
      break;
    case CONFIG_FECHA_HORA:
      lcd.setCursor(13,1);
      break;
    case CONFIG_FECHA_MINUTO:
      lcd.setCursor(16,1);
      break;
  }
  lcd.cursor();
}

void pantallaConfigMegafonia(DateTime tiempo){
  String hora = String(tiempo.hour());
  String minuto = String(tiempo.minute());

  if(tiempo.hour() < 10){
    hora = "0" + String(tiempo.hour());
  }

  if(tiempo.minute() < 10){
    minuto = "0" + String(tiempo.minute());
  }

  lcd.setCursor(0,0);
  lcd.print("Conf. Megafonia:");
  lcd.setCursor(2,1);
  lcd.print("Hora: " + hora + ":" + minuto);

  switch(configMegafoniaPos){
    case CONFIG_MEGAFONIA_HORA:
      lcd.setCursor(8,1);
      break;
    case CONFIG_MEGAFONIA_MINUTO:
      lcd.setCursor(11, 1);
      break;
  }
  lcd.cursor();
}

void pantallaConfigPortal(DateTime abrir, DateTime cerrar){
  String horaAbrir = String(abrir.hour());
  String minutoAbrir = String(abrir.minute());
  String horaCerrar = String(cerrar.hour());
  String minutoCerrar = String(cerrar.minute());
  
  if(abrir.hour() < 10){
    horaAbrir = "0" + String(abrir.hour());
  }

  if(abrir.minute() < 10){
    minutoAbrir = "0" + String(abrir.minute());
  }

  if(cerrar.hour() < 10){
    horaCerrar = "0" + String(cerrar.hour());
  }

  if(cerrar.minute() < 10){
    minutoCerrar = "0" + String(cerrar.minute());
  }

  lcd.setCursor(0,0);
  lcd.print("Conf. Portal:");
  lcd.setCursor(2,1);
  lcd.print("Hora Abrir: " + horaAbrir + ":" + minutoAbrir);
  lcd.setCursor(2,2);
  lcd.print("Hora Peche: " + horaCerrar + ":" + minutoCerrar);

  switch(configPortalPos){
    case CONFIG_PORTAL_HORA_ABRIR:
      lcd.setCursor(14,1);
      break;
    case CONFIG_PORTAL_MINUTO_ABRIR:
      lcd.setCursor(17, 1);
      break;
    case CONFIG_PORTAL_HORA_CERRAR:
      lcd.setCursor(14,2);
      break;
    case CONFIG_PORTAL_MINUTO_CERRAR:
      lcd.setCursor(17, 2);
      break;
  }
  lcd.cursor();
}

void pantallaConfigDiasSemana(){
  lcd.setCursor(0,0);
  lcd.print(" D  L  M  X  J  V  S");
  lcd.setCursor(0,1);

  for(char i = 0; i < 7; i++){
    char j = i * 3 + 1;
    lcd.setCursor(j, 1);
    lcd.print(configDiasSemanaArray[i]);
  }

  String arrow = " ";
  if(configDiasSemanaPos == 7){
    lcd.setCursor(18, 1);
    lcd.print(" ");
    arrow = ">";
  }
  lcd.setCursor(0,3);
  lcd.print(arrow + " Confirmar");

  if(configDiasSemanaPos < 7){
    if(configDiasSemanaPos == 0){
      lcd.setCursor(configDiasSemanaPos, 1);
      lcd.print(">");
      lcd.setCursor(3, 1);
      lcd.print(" ");
    }
    else {
      lcd.setCursor(configDiasSemanaPos * 3 - 3, 1);
      lcd.print(" ");
      lcd.setCursor(configDiasSemanaPos * 3, 1);
      lcd.print(">");
      lcd.setCursor(configDiasSemanaPos * 3 + 3, 1);
      lcd.print(" ");
    }
  }
}

void doOperation(DateTime actual, DateTime abrirPorta, DateTime cerrarPorta, DateTime megafonia){ 
  if (digitalRead(PIN_ESTADO_SISTEMA) == HIGH) {
   mp3_stop();
   digitalWrite(PIN_PILOTO_HORARIO, HIGH);
   digitalWrite(PIN_SALIDA_MEGAFONIA, HIGH);
   estaMegafoniaFuncionando = false;
   return; 
  }
  else if(estaPortaAberta) {
    digitalWrite(PIN_PILOTO_HORARIO, LOW);
  }
  
  bool hacerOperacion = diasSemana >> actual.dayOfTheWeek() & 1;

  unsigned long currentTime = millis();
  if(!hacerOperacion){
      if(estaPortaAberta){
        if(!tempTime && currentTime - previousTimeOperation >= 5000){
        Serial.println("Cerrar Porta: Activar Rele");
        previousTimeOperation = currentTime;
        digitalWrite(PIN_SALIDA_PORTAL, LOW);
        tempTime = true;
      }
      else if (tempTime && currentTime - previousTimeOperation >= 5000){
        Serial.println("Cerrar Porta, Megafonia: Desactivar Rele");
        digitalWrite(PIN_SALIDA_PORTAL, HIGH);
        digitalWrite(PIN_PILOTO_HORARIO, HIGH);
        digitalWrite(PIN_SALIDA_MEGAFONIA, HIGH);
        estaMegafoniaFuncionando = false;
        estaPortaAberta = false;
        tempTime = false;
      }
    }
    return;
  }

  int minutosActuales = actual.hour() * 60 + actual.minute();
  int minutosAbrirPorta = abrirPorta.hour() * 60 + abrirPorta.minute();
  int minutosCerrarPorta = cerrarPorta.hour() * 60 + cerrarPorta.minute();
  int minutosMegafonia = megafonia.hour() * 60 + megafonia.minute();
  if(!estaPortaAberta && minutosAbrirPorta - minutosActuales <= 0 && minutosCerrarPorta - minutosActuales > 0){
    //Comprobar se realmente a porta esta pechada
    if(!tempTime && currentTime - previousTimeOperation >= 5000){
      Serial.println("Abrir Porta: Activar Rele");
      previousTimeOperation = currentTime;
      digitalWrite(PIN_SALIDA_PORTAL, LOW);
      digitalWrite(PIN_PILOTO_HORARIO, LOW);
      tempTime = true;
    }
    else if (tempTime && currentTime - previousTimeOperation >= 5000) {
      Serial.println("Abrir Porta: Desactivar Rele");
      digitalWrite(PIN_SALIDA_PORTAL, HIGH);
      estaPortaAberta = true;
      tempTime = false;
    }
  }
  else if(estaPortaAberta && minutosCerrarPorta - minutosActuales <= 0 && minutosAbrirPorta - minutosActuales < 0 ){
    //Comprobar se a porta esta realmente aberta
    if(!tempTime && currentTime - previousTimeOperation >= 5000){
      Serial.println("Cerrar Porta: Activar Rele");
      previousTimeOperation = currentTime;
      digitalWrite(PIN_SALIDA_PORTAL, LOW);
      tempTime = true;
    }
    else if (tempTime && currentTime - previousTimeOperation >= 5000){     
      digitalWrite(PIN_SALIDA_PORTAL, HIGH);
      digitalWrite(PIN_PILOTO_HORARIO, HIGH);
      digitalWrite(PIN_SALIDA_MEGAFONIA, HIGH);
      estaMegafoniaFuncionando = false;
      estaPortaAberta = false;
      tempTime = false;
    }
  }

  if(!estaMegafoniaFuncionando && minutosMegafonia - minutosActuales <= 0 && minutosCerrarPorta - minutosActuales > 0){
    if(!tempTime && currentTime - previousTimeOperation >= 5000){
      Serial.println("Megafonia: Encender Rele");
      previousTimeOperation = currentTime;
      digitalWrite(PIN_SALIDA_MEGAFONIA, LOW);
      tempTime = true;
    }
    else if(tempTime && currentTime - previousTimeOperation >= 5000){
      Serial.println("Megafonia: Sonar Musica");
      mp3_play(1);
      estaMegafoniaFuncionando = true;
      tempTime = false;
      previousTimeOperation = currentTime;
    }
  }
  else if(estaMegafoniaFuncionando){
    mp3_get_state();
    int state = mp3_wait_state();
    if(state == 255 && currentTime - previousTimeOperation >= 300000) { //Repetir cada 5 min 
      mp3_play(1);     
    }
    else if (state != 255) {
      previousTimeOperation = currentTime;
    }
  }
}

void comprobarCambioHorario(DateTime fecha){
  if(!cambiouseHorario && fecha.month() == 3 && fecha.dayOfTheWeek() == 0 && (fecha.day() + 7) > 31 && fecha.hour() == 3 && fecha.minute() == 0){
    rtc.adjust(DateTime(fecha.year(), fecha.month(), fecha.day(), fecha.hour() + 1, fecha.minute(), fecha.second()));
    cambiouseHorario = true;
    EEPROM.write(7, (byte) cambiouseHorario);
  }
  else if(!cambiouseHorario && fecha.month() == 10 && fecha.dayOfTheWeek() == 0 && (fecha.day() + 7) > 31 && fecha.hour() == 3 && fecha.minute() == 0){
    rtc.adjust(DateTime(fecha.year(), fecha.month(), fecha.day(), fecha.hour() - 1, fecha.minute(), fecha.second()));
    cambiouseHorario = true;
    EEPROM.write(7, (byte) cambiouseHorario);
  }
  else if(fecha.dayOfTheWeek() == 1 && cambiouseHorario){
    cambiouseHorario = false;
    EEPROM.write(7, (byte) cambiouseHorario);
  }
}



