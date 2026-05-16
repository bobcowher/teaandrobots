---
title: "Sovol SV08 Active Cooling Mod"
date: 2026-05-16
tags: ["3d-printing", "sovol", "sv08", "cooling"]
draft: false
---

<p><strong>Download the STL files on Printables:</strong> <a href="https://www.printables.com/model/1723294-sovol-sv08-active-cooling-mod" target="_blank" rel="noopener noreferrer">Sovol SV08 Active Cooling Mod</a></p>

<p>I think THE biggest missing feature with the Sovol SV08 (once you have the enclosure) is the lack of an intake fan. There's an exhaust fan, but no intake, so you're stuck leaving the enclosure open when printing PLA.&nbsp;&nbsp;</p>
<p>Now, at a high level, what we're going to do is -</p>
<ol>
<li><p>Print a fan cutout template and an interior fan cover. You could skip the fan cover, but it stops things from accidentally hitting the fan.</p></li>
<li><p>Use the fan cutout to trace the spot for the new fan.&nbsp;</p></li>
<li><p>Cut out a spot for the new fan(preferably far away from the exhaust). </p></li>
<li><p>Wire the two fans together</p></li>
<li><p>Update Orca Slicer to turn the fans on at 80% during PLA prints</p></li>
</ol>
<p>What you need -</p>
<ul>
<li><p>An 80mm 24-volt fan with 2 pins and PWM control. I used the 80x15mm version of this one. You could also use the 80x25mm, just buy longer screws&nbsp;<a target="_blank" rel="noopener noreferrer nofollow" class="editor-link" href="https://www.amazon.com/dp/B07RY7HC7W">https://www.amazon.com/dp/B07RY7HC7W</a></p></li>
<li><p>A metal-friendly 3/16 drill bit. Black oxide works fine.&nbsp;</p></li>
<li><p>A way to cut a circular hole. I used a dremel tool and these metal cutting disks -&nbsp;<a target="_blank" rel="noopener noreferrer nofollow" class="editor-link" href="https://www.lowes.com/pd/Dremel-EZ-Lock-5-Piece-Fiber-1-1-2-in-Cutting-Wheel-Accessory/1207875">https://www.lowes.com/pd/Dremel-EZ-Lock-5-Piece-Fiber-1-1-2-in-Cutting-Wheel-Accessory/1207875</a></p></li>
<li><p>A splitter, or a way to splice some wires into a splitter. I used my soldering iron and some 2-pin Dupont connectors, but they aren't required.&nbsp;</p></li>
<li><p>4x M4-.70x30 machine screws w/m4 nuts.&nbsp;</p></li>
</ul>
<p><strong>Step 0</strong></p>
<p>Unplug the printer. We're dealing with electronics. Don't get shocked.&nbsp;</p>
<p><strong>Step 1</strong></p>
<p>Print out the attached STL files. One should be the interior fan cover, the other should be the cutout. I used PCTG because I was experimenting with it, but there's no reason you couldn't use PLA.</p>
<figure class="image image_resized image-style-align-center" data-width="75%" style="width: 75%; --image-width: 75%;"><img class="editor-image-resize" src="https://media.printables.com/media/prints/1723294/rich_content/628265f3-e121-430c-9d7b-60e43bd3eaed/image9.jpeg#%7B%22uuid%22%3A%2224561227-6c91-4791-98e2-f25ae2f01da3%22%2C%22w%22%3A2142%2C%22h%22%3A2856%7D"></figure>
<p><strong>Step 2</strong></p>
<p>Use the cutout template and a marker to trace a hole, preferably far away from the exhaust fan.&nbsp;</p>
<p><strong>Step 3</strong></p>
<p>Drill out the bolt holes and use your Dremel tool to cut out the fan hole. Please use proper eye/ear protection. When you're done, I recommend a grinding attachment to make sure there are no sharp edges, and a vacuum to clean out all of the dust &amp; metal bits before you go back to printing. End result should look something like this -</p>
<p></p>
<figure class="image image_resized image-style-align-center" data-width="75%" style="width: 75%; --image-width: 75%;"><img class="editor-image-resize" src="https://media.printables.com/media/prints/1723294/rich_content/4691692b-f58f-4788-a020-47f128b8ed77/image6.jpeg#%7B%22uuid%22%3A%22257e80c8-e36a-45ec-88d3-3aa032416fbc%22%2C%22w%22%3A2856%2C%22h%22%3A2142%7D"></figure>
<p><strong>Step 4</strong></p>
<p>Next step, install the fan by placing the plastic fan cover on the inside of the printer with the fan on the outside facing in. Put the metal cover on the outside of the fan. Results should look something like this -</p>
<p></p>
<figure class="image image_resized image-style-align-center" data-width="75%" style="width: 75%; --image-width: 75%;"><img class="editor-image-resize" src="https://media.printables.com/media/prints/1723294/rich_content/f2ba0295-68a6-4335-a969-fab26f07b69e/image1-2.jpeg#%7B%22uuid%22%3A%22619f28da-05e4-4dca-b6f4-a607c0b94794%22%2C%22w%22%3A2856%2C%22h%22%3A2142%7D"></figure>
<p></p>
<p><strong>Step 5</strong></p>
<p>Now the wiring! There's something that looks like an extra fan header on the SV08s mainboard, but it's not actually available from Klipper(at least, not that I can find). It's labeled LED1, and seems to be tied to the LED system. The next-best option is to drive both the intake and the exhaust fans together. For that, we need to split the wire.&nbsp;</p>
<p>In my case, I chose to make a splitter with my soldering iron &amp; Dupont connectors so I could easily move things around later. You could also just wire things up directly in the back. The end goal is simple though, split the wire pair going to the exhaust fan into two separate pairs of wires going to the two fans. Make sure not to accidentally cross over the red &amp; black wires as you're moving things around.&nbsp;</p>
<p>My splitter -</p>
<p></p>
<figure class="image image_resized image-style-align-center" data-width="75%" style="width: 75%; --image-width: 75%;"><img class="editor-image-resize" src="https://media.printables.com/media/prints/1723294/rich_content/a44727cc-4bf0-41a4-b0c9-3f818d8778c7/image3.jpeg#%7B%22uuid%22%3A%2203e6c9cd-9e76-48ac-bfbb-9868ab84c0ae%22%2C%22w%22%3A2142%2C%22h%22%3A2856%7D"></figure>
<p></p>
<p>All wired up -&nbsp;</p>
<p></p>
<figure class="image image_resized image-style-align-center" data-width="75%" style="width: 75%; --image-width: 75%;"><img class="editor-image-resize" src="https://media.printables.com/media/prints/1723294/rich_content/9b48a41f-85cf-4e63-b568-bcb17fa10803/image0-8.jpeg#%7B%22uuid%22%3A%223143d780-63d4-442b-8d26-ba43cfd10cc1%22%2C%22w%22%3A2142%2C%22h%22%3A2856%7D"></figure>
<p></p>
<p><strong>Step 6</strong></p>
<p>Finally, to Orca! Inside Orca, click the three little dots next to your filament, and click Edit. Inside there, you should find a Cooling tab. At the very bottom of the cooling tab, you should see an Exhaust section. Configure this as appropriate for your material and climate. Be sure to leave "Complete print" at 0, or it'll just keep spinning forever.&nbsp;</p>
<figure class="image image-style-align-center"><img class="editor-image-resize" src="https://media.printables.com/media/prints/1723294/rich_content/f1dca0f5-4b21-4d14-a509-0064bf88f675/image.png#%7B%22uuid%22%3A%2293338844-0f34-4c6f-a929-96c579baf606%22%2C%22w%22%3A1254%2C%22h%22%3A1016%7D"></figure>
<p>And that's it! Hopefully this is useful for someone else.&nbsp;</p>
