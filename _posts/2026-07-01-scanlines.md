---
layout: post
title: Scanlines Fixed
---

Oh how I've seen my share of bad scalars for old 2D games -- from exaggerated CRT effects that make me feel as though I'm on LSD to interpolation routines that make every game look like its made with papercraft. Old school computers present a problem when being displayed on modern televisions that have resolutions significantly higher than the source material. Today, I provide my solution for this.

I want to start with the output:
![image](/images/Flashback_Anim.png)
￼
My scanlines achieve three important things
- they can correct for any aspect ratio
- they show the artwork in the manner it was intended
- they do **NOT** dim the image

And it's that last one that actually helps enhance detail, bring back some of the image lost in the murkiness of the dark greys and browns. The warning stripes look like diagonal lines now, not staircases and the vines wrap around and look more organic, not like they're made from LEGO.

And the weird thing is that this is not mathematically hard to achieve and can be used for any mask effect, not just scanlines. Want a Sony Triniton look instead? This algorithm will still work and you'll still get full brigness on your screen without fakery like bloom, pushing brightness or, god forbid, injecting HDR.

The rules are simple.

## 1. Integer Vertical Scaling

This algorithm can scale from 2x up; there's no practical limit, but it's important that vertical scaling is always in integer increments. This is not widely disputed anymore, so I'll move on.

## 2. Bilinear Horizontal Scaling

On the Amiga, as an example, PAL and NTSC were not square pixels and regardless of which one you grew up with, modern displays using integer scaling in BOTH directions WILL get this wrong. This is an image of Workbench 2.0 correctly adjusted for PAL. To achieve this, the horizontal resolution needs to be pushed out slightly to 1,332 pixels instead of 1,280.

![image](/images/Workbench_KSL_PAL_Correct.png)

This looks so right to me, I can taste the nostalgia.

## 3. Preserve the Brightness

The average of the bright and dark lines should always equal the colour of the original image. To achieve this simply we use two transfer functions converting the input RGB to the output RGB we display.

### Dark Lines

For dark lines, we output each channel as a simple C' = MAX(0, C * 2 - 255). In Verilog this looks like:
```v
red_out <= { t_red[2:0], 1'b1 } & {4{t_red[3]}};
grn_out <= { t_grn[2:0], 1'b1 } & {4{t_grn[3]}};
blu_out <= { t_blu[2:0], 1'b1 } & {4{t_blu[3]}};
```
And maps like this:
```
OUT
 F                               *
 E                               
 D                             * 
 C                               
 B                           *   
 A                               
 9                         *     
 8                               
 7                       *       
 6                               
 5                     *         
 4                               
 3                   *           
 2                               
 1                 *             
 0 * * * * * * * *               
   0 1 2 3 4 5 6 7 8 9 A B C D E F	IN
```
### Light Lines

For light lines, we output each channel as a simple C' = MIN(255, C * 2). In Verilog this looks like:
```v
red_out <= { t_red[2:0], 1'b0 } | {4{t_red[3]}};
grn_out <= { t_grn[2:0], 1'b0 } | {4{t_grn[3]}};
blu_out <= { t_blu[2:0], 1'b0 } | {4{t_blu[3]}};
```
And maps like this:
```
OUT
 F                 * * * * * * * *
 E               *                
 D                                
 C             *                  
 B                                
 A           *                    
 9                                
 8         *                      
 7                                
 6       *                        
 5                                
 4     *                          
 3                                
 2   *                            
 1                                
 0 *                              
   0 1 2 3 4 5 6 7 8 9 A B C D E F	IN
```

## Final Thoughts

It's odd how much better everything looks dispite this not being a "true" CRT filter. It's nice not losing brightness on a display technology that today still struggles to achieve the same room-filling illumination of a CRT, so why make it worse. This isn't something that takes a lot of math or filtering and would add absolutely no delay to the pixel pipeline. On something like a RetroTINK, you would not have any change in the latency.

I've already implemented this on my fork of MiniMig which I'll be putting up on github shortly. I've also fixed Paula to return her to her crunchy glory and fixed some of the more eggregious wastes of FPGA logic, in part, thanks to the Lisa schematics. At least one mystery was solved -- how the colour table works on Denise. No it's not dual ported, and no it's not banked.

Anyway, cheers and next post will be an update on Buffee. I have thoughts on that, too.