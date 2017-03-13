# PICAS-Y-FIJAS
/*
 * SisPro_Proyecto1[Picas_Fijas].c
 *
 * Created: 04/03/2017 05:15:21 p.m.
 * Author : MATEO GOMEZ &  FABIAN PINZON
 */ 

/******************* Inclusión de Librerías *******************/
#include <avr/io.h>
#include "uart.c"

/******************* Definición de Estados *******************/
#define INICIO 0
#define VERIFICAR 1
#define ADIVINAR_CONTESTAR 2
#define ESPERAR 3

#define pos_PERDERJUEGO 2 //posicion auxiliar para verificar en "vectorENVIAR" si mando 4 fijas; ya que si lo hago perdi el juego

#define LEDPORT PORTB // Máscara que define TODO el Puerto_B de los LEDs

#define LED0 0x10 // Máscara para utilizar el LED0 del PuertoB
#define LED1 0x20 // Máscara para utilizar el LED1 del PuertoB
#define LED2 0x40 // Máscara para utilizar el LED2 del PuertoB
#define LED3 0x80 // Máscara para utilizar el LED3 del PuertoB

#define Pulsador0 0x01 // Máscara para el uso del Pulsador_0 del Puerto_E (CS0)
#define Pulsador3 0x08 // Máscara para el uso  del Pulsador_3 - RESET - del Puerto_E (CS3)
#define PORTE_PULLUP_CONF 0x18 // Máscara para config. el Pulsador_0 con Resistencia Pull-Up

/******************* Definición ESTRUCTURAS para recibir y enviar una RESPUESTA *******************/
typedef struct{
	    char vectorENVIAR[8]; 
	    char vectorRECIBIR[8]; 
}TRANSMISION;

typedef struct {
	    int nfijas; 
		int npicas;
	    char numeroSECRETO[4]; 
}NUMERO_SECRETO;

void picas(NUMERO_SECRETO *secreto, TRANSMISION *datos) ;
void fijas(NUMERO_SECRETO *secreto, TRANSMISION *datos );
void recibirCARACTERES(TRANSMISION *datos);
void guardarRESPUESTA2(NUMERO_SECRETO*secreto, TRANSMISION *datos);
void guardarRESPUESTA1(NUMERO_SECRETO*secreto, TRANSMISION *datos);
void apagarLEDS();
int main(void) {
    /* VARIABLES DEL MAIN */
	int estado = INICIO; //Estado INICIAL
	TRANSMISION datos;
	datos.vectorENVIAR[0]='0';
	datos.vectorENVIAR[1]='p';
	datos.vectorENVIAR[2]='0';
	datos.vectorENVIAR[3]='f';
	datos.vectorENVIAR[4]='0';
	datos.vectorENVIAR[5]='0';
	datos.vectorENVIAR[6]='0';
	datos.vectorENVIAR[7]='0';
	datos.vectorRECIBIR[0]='0';
	datos.vectorRECIBIR[1]='p';
	datos.vectorRECIBIR[2]='0';
	datos.vectorRECIBIR[3]='f';
	datos.vectorRECIBIR[4]='0';
	datos.vectorRECIBIR[5]='0';
	datos.vectorRECIBIR[6]='0';
	datos.vectorRECIBIR[7]='0';
	NUMERO_SECRETO secreto;
	secreto.nfijas = 0;
	secreto.npicas = 0;
	secreto.numeroSECRETO[0]= '5';
	secreto.numeroSECRETO[1]= '8';
    secreto.numeroSECRETO[2]= '2';
    secreto.numeroSECRETO[3]= '6';
	PORTE.DIR &= Pulsador0 ; //Se declara PULSADOR como salida   
	PORTE.PIN0CTRL |= PORTE_PULLUP_CONF ; //Se config. la resistencia Pull-Up al Pulsador_0 (CS0)
	PORTE.DIR &= Pulsador3 ; //Se declara PULSADOR como salida
	PORTE.PIN3CTRL |= PORTE_PULLUP_CONF ; //Se config. la resistencia Pull-Up al Pulsador_3 (CS3)
	
	LEDPORT.DIRSET = 0xf0 ; //TODOS los LEDs se configuran como SALIDAS
	LEDPORT.OUTSET = 0xf0 ; //TODOS los LEDs empiezan en ALTO
	TCC0.PER = 0x5B8E ;
	TCC0.CTRLA = 0x06 ;
	initUART();
	
    while(1) {
		switch(estado){
			case INICIO:
						PORTB_OUT &= ~(LED0) ;
						PORTB_OUT &= ~(LED1) ;
						PORTB_OUT &= ~(LED2) ;
						PORTB_OUT &= ~(LED3) ;
						 
						//Aquí se colocan las condiciones de TRANSICIÓN entre Estados
						if((PORTE.IN & Pulsador0)==0){ //Se pulsó el Pulsador0 para iniciar el juego
							estado = ADIVINAR_CONTESTAR ;
						}
						if(USARTC0_STATUS & USART_RXCIF_bm){ //Linea para quitar el Bloqueo - BUG de la Librería
						 	estado = VERIFICAR ; //Me llega una solicitud por UART por lo que procedo a VERIFICAR con Picas&Fijas
						}
						break;
						
			case VERIFICAR:
						apagarLEDS() ; // Reinicia los LEDs en APAGADO		
						PORTB.OUT &= ~(LED0) ;
						PORTB.OUT &= ~(LED3) ;
						
						//Aquí se utiliza el código de PICAS&FIJAS
						recibirCARACTERES(&datos);
						picas(&secreto,&datos);
						fijas(&secreto,&datos);
						//Termina código de PICAS&FIJAS
						guardarRESPUESTA2(&secreto,&datos);
						
						
						
						//Aquí se colocan las condiciones de TRANSICIÓN entre Estados
						if((PORTE.IN & Pulsador3)==0) //Se presionó el Pulsador de RESET
						estado = INICIO ;
						if(datos.vectorENVIAR[pos_PERDERJUEGO] != '4') //PASO A ADIVINAR EL NÚMERO DEL CONTRARIO
						estado = ADIVINAR_CONTESTAR ;
						if(datos.vectorENVIAR[pos_PERDERJUEGO] == '4') //PERDÍ EL JUEGO
						estado = ESPERAR ;
						break;
			
			case ADIVINAR_CONTESTAR:		
						apagarLEDS() ; //Reinicia los LEDs en APAGADO
						PORTB.OUT &= ~(LED1) ;
						PORTB.OUT &= ~(LED2) ;
						sendChar('N') ;
						
						//Aquí se utiliza el código para ADIVINAR NUMERO CONTRARIO
					    //adivinarNUMERO();
					
					
					
						
						//Aquí se colocan las condiciones de TRANSICIÓN entre Estados
						if((PORTE.IN & Pulsador3)==0) //Se presionó el Pulsador de RESET
							estado = INICIO;
						
						//AQUÍ DEBO ENVIAR POR UART MI RESPUESTA & MI INTENTO
						
						//estado = ESPERAR ;
						
						break;
			
			case ESPERAR:
						apagarLEDS() ; //Reinicia los LEDs en APAGADO
						
						//Aquí se colocan las condiciones de TRANSICIÓN entre Estados
						if((PORTE.IN & Pulsador3)==0) //Se presionó el Pulsador de RESET
							estado = INICIO;
						if(USARTC0.STATUS & USART_RXCIF_bm)
							estado = VERIFICAR ;
						/*
						if(){
							SI LO QUE LLEGA POR SOLICITUD ES 0P4F (GANÉ)
						}
						*/
						if((TCC0.INTFLAGS & 0x01)!=0){
							TCC0.INTFLAGS |= 0x01 ;
							estado = INICIO ;						
						}
						break;		
		}
	}
}


/************************ FUNCIONES DE PRUEBA ************************/
//	A)) PICAS & FIJAS:
void picas(NUMERO_SECRETO*secreto, TRANSMISION*datos) {
	int i=0, j=0;
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==1
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==2
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==3
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j=0 ; // J==4 [REINICIO] <------------------------------------
	i++ ; // i==1 ;
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==1
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==2
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==3
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j=0 ; // J==4 [REINICIO] <------------------------------------
	i++ ; // i==2 ;
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==1
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==2
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==3
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j=0 ; // J==4 [REINICIO] <------------------------------------
	i++ ; // i==3 ;
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==1
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==2
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j++ ; // j==3
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i!=j))
	secreto -> npicas++ ;
	j=0 ; // J==4 [REINICIO] <------------------------------------
	i=0 ; // I==4 [REINICIO] <------------------------------------
}

/*-----------------------------------------------------------------------------*/
void fijas(NUMERO_SECRETO *secreto, TRANSMISION *datos ) {
	int i=0, j=0;
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==1
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==2
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==3
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j=0 ; // J==4 [REINICIO] <------------------------------------
	i++ ; // i==1 ;
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==1
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==2
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==3
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j=0 ; // J==4 [REINICIO] <------------------------------------
	i++ ; // i==2 ;
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==1
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==2
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==3
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j=0 ; // J==4 [REINICIO] <------------------------------------
	i++ ; // i==3 ;
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==1
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==2
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j++ ; // j==3
	if((secreto -> numeroSECRETO[i] == datos -> vectorRECIBIR[j]) && (i==j))
	secreto -> nfijas++ ;
	j=0 ; // J==4 [REINICIO] <------------------------------------
	i=0 ; // I==4 [REINICIO] <------------------------------------
}

/*-----------------------------------------------------------------------------*/

//	B)) RECIBIR JUGADA:
void recibirCARACTERES(TRANSMISION *datos){
	int i=4 ;
	char aux_RECIBIR ='0' ;
	if(USARTC0_STATUS & USART_RXCIF_bm){ //Linea para quitar el Bloqueo - BUG de la Librería
		aux_RECIBIR = usart_receiveByte() ;
	}
	datos->vectorRECIBIR[i] = aux_RECIBIR ;
	i++ ; //i==5
	if(USARTC0_STATUS & USART_RXCIF_bm){ //Linea para quitar el Bloqueo - BUG de la Librería
		aux_RECIBIR = usart_receiveByte() ;
	}
	datos->vectorRECIBIR[i] = aux_RECIBIR ;
	i++ ; //i==6
	if(USARTC0_STATUS & USART_RXCIF_bm){ //Linea para quitar el Bloqueo - BUG de la Librería
		aux_RECIBIR = usart_receiveByte() ;
	}
	datos->vectorRECIBIR[i] = aux_RECIBIR ;
	i++ ; //i==7
	if(USARTC0_STATUS & USART_RXCIF_bm){ //Linea para quitar el Bloqueo - BUG de la Librería
		aux_RECIBIR = usart_receiveByte() ;
	}
	datos->vectorRECIBIR[i] = aux_RECIBIR ;
	i=4 ; //<----------------- RESETEO DE LA VARIABLE
}

//	C)) ENVIAR JUGADA:
/*void guardarRESPUESTA1(NUMERO_SECRETO*secreto, TRANSMISION *datos){
	int i;
	i=4;
	datos->vectorENVIAR[i];//luis 
	i=5;
	datos->vectorENVIAR[i];//luis 
	i=6;
	datos->vectorENVIAR[i];//luis 
	i=7;
	datos->vectorENVIAR[i];//luis 
	
}*/
void guardarRESPUESTA2(NUMERO_SECRETO*secreto, TRANSMISION *datos){
	int i;
	i=0 ;
	datos->vectorENVIAR[i] = secreto->npicas ;
	i=2 ;
	datos->vectorENVIAR[i] = secreto->nfijas ;
}

/*void adivinarNUMERO(){
	
	int turnos=1;
	int spf1=0, spf2=0, spf3=0;
	int anteriores=0, actuales=0;
	char enviar[5];
	char enviar1[5] = {'0','1','2','3','\0'};
	char enviar2[5] = {'4','5','6','7','\0'};
	char enviar3[5] = {'8','9','0','1','\0'};
	char temp;
	if(turnos == 1){
		guardarRESPUESTA1(&secreto,&datos);//Este es el numero que se envía en el primer turno
	}
	if(turnos == 2 && spf1 != 4){
		guardarRESPUESTA1(&secreto,&datos);//Este es el numero que se envía en el segundo turno
	}
	if(turnos == 3 && spf2 != 4){
		guardarRESPUESTA1(&secreto,&datos);//En el tercer numero enviado manda 2 caracteres x para que no se repitan las cifras enviadas en anteriores turnos

	}
	enviar[5] = '\0';
	

	if(turnos == 1){
		spf1 = picas+fijas;
	}
	if(turnos == 2){
		spf2 = picas+fijas;
	}
	if(turnos == 3){
		spf3 = picas+fijas;
	}
	turnos++;


	if((picas + fijas)== 4){

		
		if(turnos == 1){
			temp = enviar1[0];
			enviar[0] = enviar1[1];
			enviar[1] = enviar1[2];
			enviar[2] = enviar1[3];
			enviar[3] = temp;
			enviar[4] = '\0';
		}
		if(turnos == 2){
			temp = enviar1[0];
			enviar[0] = enviar1[1];
			enviar[1] = enviar1[2];
			enviar[2] = enviar1[3];
			enviar[3] = temp;
			enviar[4] = '\0';
		}


		if(fijas == 4){
			 //Esto no va aqui, solo quiero afirmar que el codigo termina aqui
		}
	}

	if(turnos > 3 && spf1 >= spf2 && spf1 >= spf2){
		enviar[0] = '4';
		enviar[1] = '1';
		enviar[2] = '2';
		enviar[3] = '3';
		anteriores = actuales;
		actuales = picas + fijas;

		if(anteriores > actuales){
			enviar[0] = '0';
			enviar[1] = '5';
			enviar[2] = '2';
			enviar[3] = '3';
			anteriores = actuales;
			actuales = picas + fijas;

			if(anteriores > actuales){
				enviar[0] = '0';
				enviar[1] = '1';
				enviar[2] = '6';
				enviar[3] = '3';
				anteriores = actuales;
				actuales = picas + fijas;

				if(anteriores > actuales){
					enviar[0] = '0';
					enviar[1] = '1';
					enviar[2] = '2';
					enviar[3] = '7';
					anteriores = actuales;
					actuales = picas + fijas;

					if(anteriores > actuales){
						enviar[0] = '8';
						enviar[1] = '1';
						enviar[2] = '2';
						enviar[3] = '3';
						anteriores = actuales;
						actuales = picas + fijas;

						if(anteriores > actuales){
							enviar[0] = '8';
							enviar[1] = '1';
							enviar[2] = '2';
							enviar[3] = '3';
							anteriores = actuales;
							actuales = picas + fijas;

							if(anteriores > actuales){
								enviar[0] = '0';
								enviar[1] = '9';
								enviar[2] = '2';
								enviar[3] = '3';
								anteriores = actuales;
								actuales = picas + fijas;
							}
						}
					}
				}
			}
		}
	}
	
}*/
void apagarLEDS(){
	LEDPORT.OUTSET = 0xf0 ; // Se coloca TODO el Puerto_B en ALTO para que los LEDs se APAGUEN (activos en BAJO)
}
