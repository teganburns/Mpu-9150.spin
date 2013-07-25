Mpu-9150.spin
=============

Mpu-9150.spin


''
''
''     QuadRotor
'' -- Jason Dorie --  
''
'' EDITED BY TEGAN BURNS  
''
''
'' MPU-3200 Gyro Module
'' Note that this code assumes an 80 MHz clock


CON
  _clkmode = xtal1 + pll16x
  _xinfreq = 5_000_000
  SCL = 0
  SDA = 1


VAR
  long PreviousTime, CurrentTime, ElapsedTime, Seconds, x, y, z, x0, y0, z0, t, temp, drift
  long Cog, Stack[32], Ready


PUB Start( ignored1, ignored2 )
  Ready := 0
  cog := cognew(MPU9150_Loop, @stack)

PUB Stop
''Stops the Cog and the PID controller
  cogstop(cog)

PUB GetTemp
  return temp

PUB GetX
  return x - x0 - drift

PUB GetY
  return y - y0 - drift

PUB GetZ
  return z - z0 - drift

PUB IsReady
  return Ready
  

PRI MPU9150_Loop | value, ack, lastTime

  x0 := y0 := z0 := 0
  x :=  y :=  z := 0
  temp := 0
  drift := 0

  outa[SCL] := 1                       ' reinitialized.  Drive SCL high.
  dira[SCL] := 1
  dira[SDA] := 0                       ' Set SDA as input

  'Give the chip some startup time to stabilize (probably unnecessary)
   waitcnt( constant(80_000_000 / 100) + cnt )
  
  ITGWriteRegisterByte (107, 1)          'Good \ PWR_MGM                    ''Internal 8MHz Oscillator
  ITGWriteRegisterByte( 27, 24)          'Good \ FS_SEL                     ''should be 2000/dps 
  ITGWriteRegisterByte(26, 2)            'Good \ DLPF_CFG                   ''should be set to the same as the ITG-Module
  ITGWriteRegisterByte( 25, 4)           'Good \ Sample rate divider        ''should be set to the same as the ITG-Module


  

  {
  InitADXL

  'ADXL test
  repeat
    ADXLStartRead( $32 )
    t := ContinueRead << 8
    t += ContinueRead
    ~~t
    x := t
    t := ContinueRead << 8
    t += ContinueRead
    ~~t
    y := t
    t := ContinueRead << 8
    t += FinishRead
    ~~t
    z := t

    'Tested at 290Hz @ 80MHz

    waitcnt( constant(80_000_000 / 200) + lastTime )
    lastTime += constant(80_000_000 / 200) 
  }

  FindZero

  Ready := 1

  'Run the main gyro reading loop
  lastTime := cnt
   
  repeat
    ITGStartRead( 65 )
    t := ContinueRead << 8
    t += ContinueRead
    ~~t
    temp := t
    t := ContinueRead << 8
    t += ContinueRead
    ~~t
    x := t
    t := ContinueRead << 8
    t += ContinueRead
    ~~t
    y := t
    t := ContinueRead << 8
    t += FinishRead
    ~~t
    z := t

    drift := (temp + 15000) / 100

    'Tested at 290Hz @ 80MHz

    waitcnt( constant(80_000_000 / 200) + lastTime )     ''i think my problem is here 
    lastTime += constant(80_000_000 / 200) 


PRI FindZero | tc, xc, yc, zc, dr

  'Find the zero points of the 3 axis by reading for ~1 sec and averaging the results
  xc := 0
  yc := 0
  zc := 0

  repeat 256
    ITGStartRead( 65 )              ''Tempature sensor Register followed by X,Y, and Z Gyros
    t := ContinueRead << 8
    t := t | ContinueRead
    ~~t
    tc := t

    t := ContinueRead << 8
    t := t | ContinueRead
    ~~t
    xc += t

    t := ContinueRead << 8
    t := t | ContinueRead
    ~~t
    yc += t
    
    t := ContinueRead << 8
    t := t | FinishRead
    ~~t
    zc += t

    dr := (tc + 15000) / 100
    xc -= dr
    yc -= dr
    zc -= dr

    waitcnt( constant(80_000_000/128) + cnt )                           ''and problem here

  if( xc > 0 )
    xc += 128
  elseif( xc < 0 )
    xc -= 128

  if( yc > 0 )
    yc += 128
  elseif( yc < 0 )
    yc -= 128

  if( zc > 0 )
    zc += 128
  elseif( zc < 0 )
    zc -= 128
    
  x0 := xc / 256
  y0 := yc / 256
  z0 := zc / 256
  
  return


PRI  ITGReadRegisterByte( addr ) : result                                                                     ''everything here should be fine
    StartSend
    WriteByte( %0110_1000 )     ' 7 bits, plus a zero bit, total of 8 bits - this is a write
    WriteByte( addr )          ' The address of the register to read from
    StartSend
    WriteByte( %0110_1001 )     ' 7 bits, plus a one bit, total of 8 bits - this is a read
    result := ReadByte( 1 )
    StopSend


PRI ITGReadRegisterWord( addr ) : result
    StartSend
    WriteByte( %0110_1000 ) 
    WriteByte( addr )
    StartSend
    WriteByte( %0110_1001 ) 
    result := ReadByte( 0 ) << 8
    result |= ReadByte( 1 )
    StopSend


PRI ITGStartRead( addr ) : result
    StartSend
    WriteByte( %0110_1000 ) 
    WriteByte( addr )
    StartSend
    WriteByte( %0110_1000 ) 


PRI ContinueRead : result
   result := 0
   dira[SDA]~                          ' Make SDA an input
  repeat 8                            ' Receive data from SDA
    outa[SCL]~~                      ' Sample SDA when SCL is HIGH
      result := (result << 1) | ina[SDA]
      outa[SCL]~
  outa[SDA] := 0                      ' Output ACK to SDA
  dira[SDA]~~                                                                                           ''Stop bit???
  outa[SCL]~~                         ' Toggle SCL from LOW to HIGH to LOW
  outa[SCL]~
  outa[SDA]~                          ' Leave SDA driven LOW


PRI FinishRead : result
   result := 0
   dira[SDA]~                          ' Make SDA an input
  repeat 8                            ' Receive data from SDA
    outa[SCL]~~                      ' Sample SDA when SCL is HIGH
      result := (result << 1) | ina[SDA]
      outa[SCL]~
  outa[SDA] := 1                      ' Output NAK to SDA
  dira[SDA]~~                                                                                               
  outa[SCL]~~                         ' Toggle SCL from LOW to HIGH to LOW
  outa[SCL]~
  outa[SDA]~                          ' Leave SDA driven LOW
 


PRI ITGWriteRegisterByte( addr , value ) : result
    StartSend
    WriteByte( %0110_1000 ) 
    WriteByte( addr )
    result := WriteByte( value )
    StopSend
                          

PRI StartSend
   outa[SCL]~~                         ' Initially drive SCL HIGH
   dira[SCL]~~
   outa[SDA]~~                         ' Initially drive SDA HIGH
   dira[SDA]~~
   outa[SDA]~                          ' Now drive SDA LOW
   outa[SCL]~                          ' Leave SCL LOW


PRI StopSend                           ' SDA goes LOW to HIGH with SCL High
   outa[SCL]~~                         ' Drive SCL HIGH
   outa[SDA]~~                         '  then SDA HIGH
   dira[SCL]~                          ' Now let them float
   dira[SDA]~                          ' If pullups present, they'll stay HIGH


                                 
PRI WriteByte( data ) : ackbit
  ackbit := 0
  data <<= 24                          ' Write 8 bits
  repeat 8
    outa[SDA] := (data <-= 1) & 1
    outa[SCL]~~                      ' Toggle SCL from LOW to HIGH to LOW
    outa[SCL]~
  dira[SDA]~                          ' Set SDA to input for ACK/NAK
  outa[SCL]~~
  ackbit := ina[SDA]                  ' Sample SDA when SCL is HIGH
  outa[SCL]~
  outa[SDA]~                          ' Leave SDA driven LOW
  dira[SDA]~~


PRI ReadByte( ackbit ) : data
'' Read in i2c data, Data byte is output MSB first, SDA data line is
'' valid only while the SCL line is HIGH.  SCL and SDA left in LOW state.
   data := 0
   dira[SDA]~                          ' Make SDA an input
  repeat 8                            ' Receive data from SDA
    outa[SCL]~~                      ' Sample SDA when SCL is HIGH
      data := (data << 1) | ina[SDA]
      outa[SCL]~
  outa[SDA] := ackbit                 ' Output ACK/NAK to SDA
  dira[SDA]~~
  outa[SCL]~~                         ' Toggle SCL from LOW to HIGH to LOW
  outa[SCL]~
  outa[SDA]~                          ' Leave SDA driven LOW


'===================================================================================================================================================================================================================
'===================================================================================================================================================================================================================
PRI InitADXL                                                                                     '' Assuming that ADXL had to do with the ITG-3200 module
  ADXLWriteRegisterByte( $2D , %1000 )



PRI ADXLWriteRegisterByte( addr , value )
    StartSend
    WriteByte( %10100110 )      'Alternate address (0x53 << 1) + 0 bit for write = 0xA6 = %10100110 
    WriteByte( addr )
    WriteByte( value )
    StopSend

PRI ADXLReadRegisterByte( addr ) : result
    StartSend
    WriteByte( %10100110 ) 
    WriteByte( addr )
    StartSend
    WriteByte( %10100111 ) 
    result := ReadByte( 1 )
    StopSend

PRI ADXLStartRead( addr ) : result
    StartSend
    WriteByte( %10100110 ) 
    WriteByte( addr )
    StartSend
    WriteByte( %10100111 ) 
        
