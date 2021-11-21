[Back](README.md)

*21 Nov 2021*

**Raycasting Game in C - 3**

![Edgar, 3, screenshot](/data/edgar003.png)

Hello,

Since the last time I've did a little bit of refactoring. Some code was moved to separate files and isolated. So there won't be a lot updates today. I've also moved majority of code not related to system to separate files and main.c should have more or less just system code now: window operations, input events, basic game loop and initialisation.

I've also added simple "shading" technique to wall rendering. It's clean and do it's job. It detects what kind of ray hit that was (horizontal or vertical) and changes texture's texel color to a bit dimmed. You could see it on a screenshot above. Here's some code that do this:

```
// Set color from sample texture.
uint32_t TexelColor = TextureBuffer[(TextureW * TextureOffsetY) + TextureOffsetX];

if (Rays[x].WasHitVertical) {
    ChangeColorIntensity(&TexelColor, WALL_SHADOW_FACTOR);
}

WriteColorBuffer(x, y, TexelColor);
```

Next time there will be some sprites, so everything will come to life a little bit.

See you!

**@HinodeCoder**
