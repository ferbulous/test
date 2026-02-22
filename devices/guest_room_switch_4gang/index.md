
detach button 1 & button 2 (so73 1)
rule1 
on button1#state=15 do power2 toggle endon
on power2#state=1 do power1 1 endon 
on power2#state=0 do power1 0 endon
on button3#state=15 do power3 toggle endon
on button4#state=15 do power4 toggle endon

#optional check the value of button state
on button2#state do publish button2 %value% endon

Assign button 2 as switch 2
detach switch 2 (so114 1)
switchmode2 12
so32 10

Add this for 
rule2 
ON system#boot DO Backlog var1 +; var2 1 ENDON
ON switch2#state=2 DO power4 TOGGLE ENDON
ON switch2#state=4 DO DIMMER %var1% ENDON
ON switch2#state=7 DO mult2 -1 ENDON
ON var2#state==-1 DO var1 - ENDON
ON var2#state==1 DO var1 + ENDON
ON dimmer#state==1 DO mult2 -1 ENDON
ON dimmer#state==100 DO mult2 -1 ENDON
ON switch2#state=2 DO publish switch2 %value% ENDON

Assign device group name for relay 1
devgroupname1 guestnightlamp

Assign 1 pin for pwm1
This auto assign to relay 5 so relay 4 (outside light need to change to output_high)
so now it's power4
