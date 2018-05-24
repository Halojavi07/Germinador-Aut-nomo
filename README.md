# Germinador-Aut-nomo
Codigo Fuente de arduino y codigos php/html

//Librearias
#include <SoftwareSerial.h>
#include <LiquidCrystal.h>
#include <DHT.h>
#include <DHT_U.h>


SoftwareSerial SerialESP8266(10,9); // RX, TX ESP8266

String server = "148.202.220.25";
int sensor = 8;
int rele = 11;
int rele2 = 12;
int on = 30;
#define DHTTYPE DHT11 
DHT dht(sensor, DHTTYPE);
LiquidCrystal lcd(2,3,4,5,6,7); //Puertos LCD


String cadena="";

void setup() {

  SerialESP8266.begin(9600);
  Serial.begin(9600);
  lcd.begin(16,2);
  dht.begin();
  pinMode(rele, OUTPUT);
  pinMode(rele2,OUTPUT);
  
  SerialESP8266.setTimeout(2000);
  
  //Verificamos si el ESP8266 responde
  SerialESP8266.println("AT");
  if(SerialESP8266.find("OK"))
    Serial.println("Respuesta AT correcto");
  else
    Serial.println("Error en ESP8266");

  //-----Configuración de red-------//

    //ESP8266 en modo estación (nos conectaremos a una red existente)
    SerialESP8266.println("AT+CWMODE=1");
    if(SerialESP8266.find("OK"))
      Serial.println("ESP8266 en modo Estacion");
      
    //Nos conectamos a una red wifi 
    SerialESP8266.println("AT+CWJAP=\"CUCI wifi2\",\"CUCI wifi2\"");//Ponemos el SSID y Password de la red a conectar
    Serial.println("Conectandose a la red ...");
    SerialESP8266.setTimeout(10000); 
    if(SerialESP8266.find("OK"))
      Serial.println("WIFI conectado");
    else
      Serial.println("Error al conectarse en la red");
    SerialESP8266.setTimeout(2000);
    //Desabilitamos las conexiones multiples
    SerialESP8266.println("AT+CIPMUX=0");
    if(SerialESP8266.find("OK"))
      Serial.println("Multiconexiones deshabilitadas");
    
  //------fin de configuracion-------------------

  delay(1000);
  
}


void loop() {
  
  //Iniciamos las variables del sensor DHT11

   int a = dht.readTemperature();
   int b = dht.readHumidity();
                 
   //Imprimimos los datos del sensor en la pantalla LCD
           lcd.setCursor(0,0);
           lcd.print("Temperatura: ");
           lcd.print(a);
           lcd.print(" *C");
           lcd.setCursor(0,1);
           lcd.print("Humedad: ");
           lcd.print(b);
           lcd.print("% (RH)");
  
if (isnan(a) || isnan(b)) {
Serial.println("Falla al leer el sensor DHT11!");
}

           
  //---------enviamos las variables al servidor---------------------
  
      //Nos conectamos con el servidor:
      
      SerialESP8266.println("AT+CIPSTART=\"TCP\",\"" + server + "\",80");
      if( SerialESP8266.find("OK"))
      {  
          Serial.println();
          Serial.println();
          Serial.println();
          Serial.println("ESP8266 conectado con el servidor...");             
    
          //encabezado de la peticion http
          String peticionHTTP= "GET /germinador/insertar.php?temp="+String(a)+"&humedad="+String(b);
          peticionHTTP=peticionHTTP+" HTTP/1.1\r\n";
          peticionHTTP=peticionHTTP+"Host: 148.202.220.25\r\n\r\n";
          

    
          //Enviamos el tamaño en caracteres de la peticion http:  
          SerialESP8266.print("AT+CIPSEND=");
          SerialESP8266.println(peticionHTTP.length());

          //esperamos a ">" para enviar la petcion  http
          if(SerialESP8266.find(">")) // ">" indica que podemos enviar la peticion http
          {
            Serial.println("Enviando HTTP . . .");
            SerialESP8266.println(peticionHTTP);
            if( SerialESP8266.find("SEND OK"))
            {  
              Serial.println("Peticion HTTP enviada:");
              Serial.println();
              Serial.println(peticionHTTP);
              Serial.println("Esperando respuesta...");
              
              boolean fin_respuesta=false; 
              long tiempo_inicio=millis(); 
              cadena="";
              
              while(fin_respuesta==false)
              {
                  while(SerialESP8266.available()>0) 
                  {
                      char c=SerialESP8266.read();
                      Serial.write(c);//Imprime la repueta del servidor en pantalla
                      cadena.concat(c);  //guardamos la respuesta en el string "cadena"
                  }
                
                  if(cadena.length()>800) 
                  {
                    Serial.println("La respuesta a excedido el tamaño maximo");
                    
                    SerialESP8266.println("AT+CIPCLOSE");
                    if( SerialESP8266.find("OK"))
                      Serial.println("Conexion finalizada");
                    fin_respuesta=true;
                  }
                  if((millis()-tiempo_inicio)>10000) //Finalizamos si ya han transcurrido 10 seg
                  {
                    Serial.println("Tiempo de espera agotado");
                    SerialESP8266.println("AT+CIPCLOSE");
                    if( SerialESP8266.find("OK"))
                      Serial.println("Conexion finalizada");
                    fin_respuesta=true;
                  }
                  if(cadena.indexOf("CLOSED")>0) //si recibimos un CLOSED significa que ha finalizado la respuesta
                  {
                    Serial.println();
                    Serial.println("Cadena recibida correctamente, conexion finalizada");         
                    fin_respuesta=true;
                  }
                  
              }
    
              
            }
            else
            {
              Serial.println("No se ha podido enviar HTTP.....");
           }            
          }
      }
      else
      {
        Serial.println("No se ha podido conectarse con el servidor");
      }

       //Encender o Apagar Ventilador
           if(a > on){
                digitalWrite(rele, LOW );
                
           }
           else if (a<=28){
                digitalWrite(rele, HIGH);
           }
         

       //Encender o Apagar Iluminacion
            if(a < 28){
                digitalWrite(rele2, LOW );
                
           }
           else if (a>=29){
                digitalWrite(rele2, HIGH);
           }

      

        
  //-------------------------------------------------------------------------------

  delay(2000); //pausa de 30seg antes de conectarse nuevamente al servidor
}
