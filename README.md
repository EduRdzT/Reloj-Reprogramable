# Reloj-Reprogramable
#### `Reloj Reprogramable con VHDL y ENSAMBLADOR`

**Índice**
- [Objetivos](#id1)
- [Marco Teórico](#id2)
- [Desarrollo Experimental](#id3)
- [Conclusiones](#id4)
- [Referencias](#id5)

## OBJETIVOS<a name="id1"></a>
  
### Objetico general

- Conocer el entorno de trabajo del software PicoBlaze y Xilinx

### Objetivos específicos 

- Realizar un código en PicoBlaze de un reloj y mostrarlo es una LCD.
- Usar en conjunto Xilinx con PicoBlaze.

## MARCO TEÓRICO<a name="id2"></a>

Los circuitos digitales, a no ser que sean asíncronos, van comandados por un reloj cuya frecuencia puede variar según el tipo de sistema digital del que se trate. Desde microprocesador 6502 que funcionaba con un reloj de 1Mhz hasta los actuales, que funcionan en el orden de los gigahercios, no han pasado ni cuatro décadas. En un sistema digital complejo es habitual que necesitemos obtener diferentes frecuencias de reloj para diferentes subsistemas. Un ejemplo muy claro puede ser el de un reloj digital que tiene que contar los segundos, por lo tanto, necesita un reloj de 1Hz (un pulso por segundo). En este artículo vamos a ver un ejemplo práctico de cómo obtener un reloj de 1Hz a partir de otro de 50Mhz en VHDL, y vamos a probarlo un una FPGA.

La técnica usada para dividir la frecuencia de un reloj es usar biestables conectados en cascada. En el siguiente esquema se detalla el funcionamiento.

![Arquitectura ALU](/Img/Imagen1.png)
 
Cada biestable divide la frecuencia a la mitad, así que la idea es ir acoplando biestables hasta obtener la frecuencia deseada. En la práctica pueden usarse contadores integrados como el 7493. En una FPGA podemos conseguir el mismo efecto usando un contador, aunque la mayoría ya trae circuitos especializados para ello, vamos a ver cómo se hace en VHDL.

## DESARROLLO EXPERIMENTAL<a name="id3"></a>

1. Códigos fuente:
   - Top VHDL
 
```VHDL
-	library IEEE;
-	use IEEE.STD_LOGIC_1164.ALL;
-	
-	entity TOP is
-		PORT(
-				CLK: IN STD_LOGIC;
-				RESET : IN STD_LOGIC;
-				SW : IN STD_LOGIC_VECTOR(7 DOWNTO 0);
-				LCD : OUT STD_LOGIC_VECTOR(7 DOWNTO 0)
-		);
-	end TOP;
-	
-	architecture Behavioral of TOP is
-	SIGNAL PORT_ID : STD_LOGIC_VECTOR(7 DOWNTO 0);
-	SIGNAL IN_PORT , DATO , OUT_PORT : STD_LOGIC_VECTOR(7 DOWNTO 0);
-	SIGNAL WRITE_STROBE : STD_LOGIC;
-	SIGNAL LED_REG : STD_LOGIC_VECTOR(7 DOWNTO 0);
-	SIGNAL ADDRESS : STD_LOGIC_VECTOR(9 DOWNTO 0);
-	SIGNAL INSTRUCTION : STD_LOGIC_VECTOR(17 DOWNTO 0);
-	SIGNAL CLK_DIV : STD_LOGIC;
-	SIGNAL READ_STROBE : STD_LOGIC;
-	
-	
-	component kcpsm3 is
-	    Port (      address : out std_logic_vector(9 downto 0);
-	            instruction : in std_logic_vector(17 downto 0);
-	                port_id : out std_logic_vector(7 downto 0);
-	           write_strobe : out std_logic;
-	               out_port : out std_logic_vector(7 downto 0);
-	            read_strobe : out std_logic;
-	                in_port : in std_logic_vector(7 downto 0);
-	              interrupt : in std_logic;
-	          interrupt_ack : out std_logic;
-	                  reset : in std_logic;
-	                    clk : in std_logic);
-	end COMPONENT;
-	
-	component memoria is
-	    Port (      address : in std_logic_vector(9 downto 0);
-	            instruction : out std_logic_vector(17 downto 0);
-	                    clk : in std_logic);
-	end COMPONENT;
-	
-	component divisor_frecuencia is
-		port (
-				clk,reset: in std_logic;
-				clk_div: inout std_logic);
-	end component;
-	
-	COMPONENT puerto_salida is
-	 
-		 Port ( --Entradas
-	           rst : in std_logic;--Restauración
-				  enable : in std_logic;--Habilitación de carga proveniente del picoBlaze
-				  direccion : in std_logic_vector(7 downto 0);--Dirección del puerto a escribir proveniente del picoBlaze
-	           dato : in std_logic_vector(7 downto 0);--Dato a grabar en el puerto
-	           --Salidas
-				  puerto_s: out std_logic_vector(7 downto 0)--Dato actual y estable en el puerto
-				);           
-	end COMPONENT;
-	
-	COMPONENT puerto is
-	    Port ( puerto_e : in std_logic_vector(7 downto 0);
-	           dato : out std_logic_vector(7 downto 0);
-				  direccion:in std_logic_vector(7 downto 0);
-	           enable : in std_logic;
-				  rst:in std_logic);
-	end COMPONENT;
-	
-	begin
-	
-		ENTRADA : puerto port map(
-		puerto_e		=>	sw,
-		dato			=>	dato,
-		direccion	=>	port_id,
-		enable		=>	read_strobe,
-		rst			=>	reset
-		);
-		
-		SALIDA : puerto_salida port map (
-		rst 			=>	reset,
-		enable		=>	write_strobe,
-		direccion	=>	port_id,
-		dato			=> out_port,
-		puerto_s		=>	lcd
-		);
-		
-		reloj : divisor_frecuencia port map (
-		clk      =>	clk,
-		reset    => reset,
-		clk_div  =>	clk_div
-		);
-		
-		IN_PORT <= dato(6 DOWNTO 0) & CLK_DIV; 
-		
-	   picoblaze_system : kcpsm3 port map(
-			address 			=> address,
-			instruction		=> instruction,
-	      port_id        => port_id,
-	      write_strobe   => write_strobe,
-			out_port       => out_port,
-	      read_strobe    => read_strobe,
-	      in_port        => in_port,
-	      interrupt      => '0',
-	      interrupt_ack  => open,
-	      reset          => reset,
-	      clk            => clk
-	   );
-		
-	   Program_memoria : memoria port map(
-	      address        => address,
-	      instruction    => instruction,
-	      clk            => clk
-	   );		
-	 
-	
-	end Behavioral;
```
   - Archivo ucf
```UCF
NET "CLK" LOC = C9;
NET "RESET" LOC = K17;
NET "LCD(7)" LOC = M15;
NET "LCD(6)" LOC = P17;
NET "LCD(5)" LOC = R16;
NET "LCD(4)" LOC = R15;
NET "LCD(3)" LOC = F9;
NET "LCD(2)" LOC = L18;
NET "LCD(1)" LOC = L17;
NET "LCD(0)" LOC = M18;
NET "SW(0)" LOC = N17;
NET "SW(1)" LOC = H18;
NET "SW(4)" LOC = L14;
NET "SW(3)" LOC = L13;
NET "SW(2)" LOC = V4;
NET "SW(5)" LOC = D18;
NET "SW(6)" LOC = H13;
```
   - Programa psm
```PSM
-	VHDL "ROM_form.vhd", "ROM_prog.vhd","memoria"
-	
-	LCD_output_port DSOUT $01 ;LCD character module output
-	PE1 DSIN $00;
-	ORG 0
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;VARIABLES;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	uni EQU S7 ;;SEGUNDOS
-	dec EQU S8 ;;SEGUNDOS
-	cen EQU S9 ;;MINUTOS
-	uni_mil EQU SA ;;MINUTOS
-	dec_mil EQU SB ;;HORA
-	cen_mil EQU SC ;;HORA
-	apo EQU SD
-	asc EQU $30
-	ban EQU SF
-	;data and control
-	ENT EQU S6
-	LCD_E EQU $01 ; active High Enable E - bit0
-	LCD_RW EQU $02 ; Read=1 Write=0 RW - bit1
-	LCD_RS EQU $04 ; Instruction=0 Data=1 RS - bit2
-	LCD_drive EQU $08 ; Master enable (active High)
-	; - bit3
-	LCD_DB4 EQU $10 ; 4-bit Data DB4 - bit4
-	LCD_DB5 EQU $20 ; interface Data DB5 - bit5
-	LCD_DB6 EQU $40 ; Data DB6 - bit6
-	LCD_DB7 EQU $80 ; Data DB7 - bit7
-	
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;INICILAIZAR;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	LOAD uni,$09
-	LOAD dec,$05
-	LOAD cen,$09
-	LOAD uni_mil,$05
-	LOAD dec_mil,$03
-	LOAD cen_mil,$02
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;INICIO;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	LOAD ban,$01
-	load apo,cen_mil
-	xor apo,$02
-	jump nz,inicio
-	load sE,$04
-	jump begin
-	inicio:
-	LOAD sE,$0A
-	;;;;;;;;;;;;;;;;;;;;;;;;;;:initialise LCD display;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	begin:
-	CALL LCD_reset ;initialise LCD display
-	
-	cold_start:
-	LOAD s5, $C0 ;Line 2 position 1
-	CALL LCD_write_inst8
-	CALL Display
-	JUMP cold_start
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;; ESCRIBIR ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	Display:
-	
-	CERO:
-	AND ban,$01
-	JUMP Z,HORAS
-	IN ENT,PE1
-	AND ENT, $02
-	jump z,MINUTOS
-	call SUMA_MINUTOS
-	MINUTOS:
-	IN ENT,PE1
-	AND ENT, $04
-	jump z,HORAS
-	CALL SUMA_HORAS
-	HORAS:
-	IN ENT,PE1
-	AND ENT,$01
-	JUMP Z,CERO
-	
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;CONTADOR;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	ADD uni,$01
-	load apo,uni
-	xor apo,$0A
-	jump nz,ascii
-	load uni,$00
-	
-	ADD dec,$01
-	load apo,dec
-	xor apo,$06
-	jump nz,ascii
-	load dec,$00
-	
-	CALL SUMA_MINUTOS
-	AND apo,$01
-	JUMP Z,ascii
-	call SUMA_HORAS
-	
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;CODIGO ASCII;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	ascii:
-	load apo,cen_mil
-	or apo,asc
-	LOAD s5, apo
-	CALL LCD_write_data
-	
-	load apo,dec_mil
-	or apo,asc
-	LOAD s5, apo
-	CALL LCD_write_data
-	
-	LOAD s5, $3A ;':'
-	CALL LCD_write_data
-	
-	load apo,uni_mil
-	or apo,asc
-	LOAD s5, apo
-	CALL LCD_write_data
-	
-	load apo,cen
-	or apo,asc
-	LOAD s5, apo
-	CALL LCD_write_data
-	
-	LOAD s5, $3A ;':'
-	CALL LCD_write_data
-	
-	load apo,dec
-	or apo,asc
-	LOAD s5, apo
-	CALL LCD_write_data
-	
-	load apo,uni
-	or apo,asc
-	LOAD s5, apo
-	CALL LCD_write_data
-	
-	UNO:
-	IN ENT,PE1
-	LOAD ban,$01
-	AND ENT,$01
-	JUMP NZ,UNO
-	RET
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;; Tiempo ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;	
-	delay_1us:
-	LOAD s0, $0B
-	wait_1us:
-	SUB s0, $01
-	JUMP NZ, wait_1us
-	RET
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;40 x 1us = 40us;;;;;;;;;;;;;;;;;;;;;;;Delay of 1ms.
-	delay_40us:
-	LOAD s1, $28 ;
-	wait_40us:
-	CALL delay_1us
-	SUB s1, $01
-	JUMP NZ, wait_40us
-	RET
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;Delay of 1ms.
-	;Registers used s0, s1, s2
-	delay_1ms:
-	LOAD s2, $19 ;25 x 40us = 1ms
-	wait_1ms:
-	CALL delay_40us
-	SUB s2, $01
-	JUMP NZ, wait_1ms
-	RET
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;Delay of 20ms.
-	;Delay of 20ms used during initialisation.
-	delay_20ms:
-	LOAD s3, $14 ;20 x 1ms = 20ms
-	wait_20ms:
-	CALL delay_1ms
-	SUB s3, $01
-	JUMP NZ, wait_20ms
-	RET
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;Delay of approximately 1 second.
-	;Registers used s0, s1, s2, s3, s4
-	delay_1s:
-	LOAD s4, $32 ;50 x 20ms = 1000ms
-	wait_1s:
-	CALL delay_20ms
-	SUB s4, $01
-	JUMP NZ, wait_1s
-	RET
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;ACTIVA EL BIT E;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	LCD_pulse_E:
-	XOR s4, LCD_E ;E=1
-	OUT s4, LCD_output_port
-	CALL delay_1us
-	XOR s4, LCD_E ;E=0
-	OUT s4, LCD_output_port
-	RET
-	
-	;Write 4-bit instruction to LCD display.
-	;
-	;The 4-bit instruction should be provided in the
-	;upper 4-bits of register s4.
-	;Note that this routine does not release the master
-	;enable but as it is only
-	;used during initialisation and as part of the
-	;8-bit instruction write it should be acceptable.
-	;Registers used s4
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;ESCRIBIR;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	LCD_write_inst4:
-	AND s4, $F8 ;Enable=1 RS=0
-	;Instruction, RW=0
-	;Write, E=0
-	OUT s4, LCD_output_port ;set up RS and RW>40ns
-	;before enable pulse
-	CALL LCD_pulse_E
-	RET
-	
-	;Write 8-bit instruction to LCD display.
-	;The 8-bit instruction should be provided in
-	; register s5.
-	;Instructions are written using the following
-	;sequence
-	; Upper nibble
-	; wait >1us
-	; Lower nibble
-	; wait >40us
-	;Registers used s0, s1, s4, s5
-	
-	LCD_write_inst8:
-	LOAD s4, s5
-	AND s4, $F0
-	OR s4, LCD_drive ;Enable=1
-	CALL LCD_write_inst4 ;write upper nibble
-	CALL delay_1us ;wait >1us
-	LOAD s4, s5 ;select lower nibble
-	;with
-	SL1 s4 ;Enable=1
-	SL0 s4 ;RS=0 Instruction
-	SL0 s4 ;RW=0 Write
-	SL0 s4 ;E=0
-	CALL LCD_write_inst4 ;write lower nibble
-	CALL delay_40us ;wait >40us
-	LOAD s4, $F0 ;Enable=0 RS=0
-	;Instruction, RW=0
-	;Write, E=0
-	OUT s4, LCD_output_port ;Release master enable
-	RET
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	LCD_write_data:
-	LOAD s4, s5
-	AND s4, $F0 ;Enable=0 RS=0
-	;Instruction, RW=0
-	;Write, E=0
-	OR s4, $0C ;Enable=1 RS=1 Data,
-	;RW=0 Write, E=0
-	OUT s4, LCD_output_port ;set up RS and RW>40ns
-	;before enable pulse
-	CALL LCD_pulse_E ;write upper nibble
-	CALL delay_1us ;wait >1us
-	LOAD s4, s5 ;select lower nibble
-	;with
-	SL1 s4 ;Enable=1
-	SL1 s4 ;RS=1 Data
-	SL0 s4 ;RW=0 Write
-	SL0 s4 ;E=0
-	OUT s4, LCD_output_port ;set up RS and RW>40ns
-	;before enable pulse
-	CALL LCD_pulse_E ;write lower nibble
-	CALL delay_40us ;wait >40us
-	LOAD s4, $F0 ;Enable=0 RS=0
-	;Instruction, RW=0
-	;Write, E=0
-	OUT s4, LCD_output_port ;Release master enable
-	RET
-	;Read 8-bit data from LCD display.
-	;The 8-bit data will be read from the current LCD
-	;memory address and will be returned in register s5.
-	;It is advisable to set the LCD address (cursor
-	;position) before using the data read for the first
-	;time otherwise the display may generate invalid data
-	;on the first read.
-	;Data bytes are read using the following sequence
-	; Upper nibble
-	; wait >1us
-	; Lower nibble
-	; wait >40us
-	;Registers used s0, s1, s4, s5
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;RESET;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	LCD_reset:
-	CALL delay_20ms ;wait more that
-	;15ms for display to be ready
-	LOAD s4, $30
-	CALL LCD_write_inst4 ;send '3'
-	CALL delay_20ms ;wait >4.1ms
-	CALL LCD_write_inst4 ;send '3'
-	CALL delay_1ms ;wait >100us
-	CALL LCD_write_inst4 ;send '3'
-	CALL delay_40us ;wait >40us
-	LOAD s4, $20
-	CALL LCD_write_inst4 ;send '2'
-	CALL delay_40us ;wait >40us
-	LOAD s5, $28 ;Function set
-	CALL LCD_write_inst8
-	LOAD s5, $06 ;Entry mode
-	CALL LCD_write_inst8
-	LOAD s5, $0C ;Display control
-	CALL LCD_write_inst8
-	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;LIMPIAR LCD;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
-	LCD_clear:
-	LOAD s5, $01 ;Display clear
-	CALL LCD_write_inst8
-	CALL delay_1ms ;wait >1.64ms for
-	;display to clear
-	CALL delay_1ms
-	RET
-	
-	SUMA_MINUTOS:
-	ADD cen,$01
-	load apo,cen
-	xor apo,$0A
-	jump nz,FIN
-	load cen,$00
-	
-	ADD uni_mil,$01
-	load apo,uni_mil
-	xor apo,$06
-	jump nz,FIN
-	LOAD apo,$01
-	load uni_mil,$00
-	JUMP MAS
-	FIN:
-	LOAD apo,$00
-	MAS:
-	LOAD ban,$00
-	RET
-	
-	SUMA_HORAS:
-	ADD dec_mil,$01
-	load apo,dec_mil
-	xor apo,sE
-	jump nz,FINAL
-	load dec_mil,$00
-	
-	ADD cen_mil,$01
-	load apo,cen_mil
-	xor apo,$02
-	jump nz,siguiente
-	load sE,$04
-	siguiente:
-	load apo,cen_mil
-	xor apo,$03
-	jump nz,FINAL
-	load cen_mil,$00
-	LOAD sE,$0A
-	FINAL:
-	LOAD ban,$00
-	RET
```
   - Diagrama RTL

![Arquitectura ALU](/Img/Imagen2.png)

## CONCLUSIONES<a name="id4"></a>

Las interrupciones fueron muy útiles ya que lo que se necesitaba era realizar varios procesos en un segundo sin tener que contar un segundo en el programa principal. También se aprendió a poner un modo de espera para poder capturar los flancos de subida del divisor de frecuencia.

Se logro obtener todos los objetivos propuestos en la practica haciendo funcionar el reloj y los switches para poder modificar la hora del reloj.

Un problema que tuvimos fue con los switch de la tarjeta ya que no funcionaban correctamente.

## REFERENCIAS<a name="id5"></a>

> https://www.digilogic.es/divisor-de-frecuencia-reloj-1hz-vhdl/   -(2017)
