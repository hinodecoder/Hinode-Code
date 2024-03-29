[Back](README.md)

*11 Nov 2021*

**Raycasting Game in C - 1**

![Corridor, 1, screenshot](/data/corridor001.png)

I’ve started this “blog” mainly because I wanted to have a record of my work on raycasting engine for game in C. I always wanted to write pure software renderer using technique from classic Wolfenstein 3D by id Software. Major problem was always time constraints that I have and long term use of this software - what kind of game I should be doing? What platforms it should be working on?

So after a while I started to coding a raycaster with a help of Pikuma courses. It is great by the way. Some time has passed and I’ve needed to give myself a goal - to be able to finish it to a degree that I could make a game using it. So I’ve found 7d-FPS game jam and wanted to try.

I don’t know if I will make it to the finish line. Game jam starts 3rd December 2021 and I need to finish that raycaster before that day. Not to mention I need to create an FPS game framework on it to be able to make an actual game.

So this is a record of what I do and let’s see if I’ll do it.

Here is code I’ve written so far based on Pikuma course. It’s just a record before I set up proper GitHub repository and make it more “multiplatform”.

***This code is a mess. Don't use it. I assure you that I'll post better code when raycaster will looks like something useful. For now it's some crazy stuff in one place.***

```
#include <stdio.h>
#include <limits.h>
#include <SDL2/SDL.h>
#include "consts.h"





const int Map[MAP_NUM_ROWS][MAP_NUM_COLS] = {
    {1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1 ,1, 1, 1, 1, 1, 1, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1},
    {1, 0, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1},
    {1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1}
};

typedef struct player_t {
    float X;
    float Y;
    float Width;
    float Height;
    int TurnDirection; // -1 left, +1 right
    int WalkDirection; // -1 back
    float RotationAngle;
    float WalkSpeed;
    float TurnSpeed;
} player_t;

// Create global player
player_t Player;



typedef struct ray_t {
    float RayAngle;
    float WallHitX;
    float WallHitY;
    float Distance;
    int WalHitVertical;
    int IsRayFacingUp;
    int IsRayFacingDown;
    int IsRayFacingLeft;
    int IsRayFacingRight;
    int WallHitContent; // For ex. color or texture index.
} ray_t;

ray_t Rays[NUM_RAYS];



SDL_Window* Window = NULL;
SDL_Renderer* Renderer = NULL;
int IsGameRunning = FALSE;
int TicksLastFrame = 0;

uint32_t* ColorBuffer = NULL;
SDL_Texture* ColorBufferTexture = NULL;

int InitializeWindow(void) {
    if (SDL_Init(SDL_INIT_EVERYTHING) != 0) {
        fprintf(stderr, "Error initializing SDL.\n");
        return FALSE;
    }

    Window = SDL_CreateWindow( NULL,
                               SDL_WINDOWPOS_CENTERED,
                               SDL_WINDOWPOS_CENTERED,
                               WINDOW_W,
                               WINDOW_H,
                               //SDL_WINDOW_BORDERLESS);
                               //SDL_WINDOW_FULLSCREEN );
                               0);

    if (!Window) {
        fprintf(stderr, "Error creating window.\n");
        return FALSE;
    }

    Renderer = SDL_CreateRenderer(Window, -1, SDL_RENDERER_PRESENTVSYNC);

    if (!Renderer) {
        fprintf(stderr, "Error creating SDL Renderer.\n");
    }

    SDL_SetRenderDrawBlendMode(Renderer, SDL_BLENDMODE_BLEND);

    return TRUE;
}

void DestroyWindow(void) {
    free(ColorBuffer);
    SDL_DestroyTexture(ColorBufferTexture);
    
    SDL_DestroyRenderer(Renderer);
    SDL_DestroyWindow(Window);
    SDL_Quit();
}


void Setup(void) {
    Player.X = WINDOW_W / 2;
    Player.Y = WINDOW_H / 2;
    Player.Width = 1;
    Player.Height = 1;
    Player.TurnDirection = 0;
    Player.WalkDirection = 0;
    Player.RotationAngle = PI / 2;
    Player.WalkSpeed = 200;
    Player.TurnSpeed = 85 * (PI / 180);
    
    // Allocate color buffer here
    ColorBuffer = (uint32_t*) malloc(sizeof(uint32_t) * (uint32_t)WINDOW_W * (uint32_t)WINDOW_H);
    
    // Create color buffer texture for SDL.
    ColorBufferTexture = SDL_CreateTexture(Renderer, SDL_PIXELFORMAT_ARGB8888, SDL_TEXTUREACCESS_STREAMING, WINDOW_W, WINDOW_H);
}


int MapHasWallAt(float X, float Y) {
    if (X < 0 || X > WINDOW_W || Y < 0 || Y > WINDOW_H) {
        return TRUE;
    }
    int MapGridIndexX = floor(X / TILE_SIZE);
    int MapGridIndexY = floor(Y / TILE_SIZE);
    return Map[MapGridIndexY][MapGridIndexX] != 0;
}


void MovePlayer(float DeltaTime) {
    Player.RotationAngle += Player.TurnDirection * Player.TurnSpeed * DeltaTime;
    float MoveStep = Player.WalkDirection * Player.WalkSpeed * DeltaTime;

    float NewPlayerX = Player.X + cos(Player.RotationAngle) * MoveStep;
    float NewPlayerY = Player.Y + sin(Player.RotationAngle) * MoveStep;

    if (!MapHasWallAt(NewPlayerX, NewPlayerY)) {
        Player.X = NewPlayerX;
        Player.Y = NewPlayerY;
    }
}


void RenderPlayer(void) {
    SDL_SetRenderDrawColor(Renderer, 255, 255, 255, 255);
    SDL_Rect PlayerRect = {
        Player.X * MINIMAP_SCALE_FACTOR,
        Player.Y * MINIMAP_SCALE_FACTOR,
        Player.Width * MINIMAP_SCALE_FACTOR,
        Player.Height * MINIMAP_SCALE_FACTOR
    };

    SDL_RenderFillRect(Renderer, &PlayerRect);

    SDL_RenderDrawLine( Renderer,
                        MINIMAP_SCALE_FACTOR * Player.X,
                        MINIMAP_SCALE_FACTOR * Player.Y,
                        MINIMAP_SCALE_FACTOR * Player.X + cos(Player.RotationAngle) * 40,
                        MINIMAP_SCALE_FACTOR * Player.Y + sin(Player.RotationAngle) * 40
                        );
}

float NormalizeAngle(float Angle) {
    Angle = remainder(Angle, TWO_PI);
    
    if (Angle < 0) {
        Angle = TWO_PI + Angle;
    }
    
    return Angle;
}

float DistanceBetweenPoints(float X1, float Y1, float X2, float Y2) {
    return sqrtf((X2 - X1) * (X2 - X1) + (Y2 - Y1) * (Y2 - Y1));
}

void CastRay(float RayAngle, int StripId) {
    // TODO: implement crazy stuff
    
    RayAngle = NormalizeAngle(RayAngle);
    
    // Angle direction
    int IsRayFacingDown = RayAngle > 0 && RayAngle < PI;
    int IsRayFacingUp = !IsRayFacingDown;
    int IsRayFacingRight = RayAngle < 0.5f * PI || RayAngle > 1.5f * PI;
    int IsRayFacingLeft = !IsRayFacingRight;
    
    float xintercept;
    float yintercept;
    float xstep;
    float ystep;

    // -----------------------------------
    // HORIZONTAL RAY-GRID INTERSECTION
    // -----------------------------------
    int foundHWallHit = FALSE;
    float wallHitX_horizontal = 0;
    float wallHitY_horizontal = 0;
    int HorizontalWallContent = 0;

    // Find the y coordinate of the closest horizontal grid intersection
    yintercept = floor(Player.Y / TILE_SIZE) * TILE_SIZE;
    yintercept += IsRayFacingDown ? TILE_SIZE : 0; // instead of using conditional just * 1 or 0.
    xintercept = Player.X + (yintercept - Player.Y) / tan(RayAngle);

    // Calculate xstep and ystep (deltas)
    ystep = TILE_SIZE;
    ystep *= IsRayFacingUp ? -1 : 1;

    xstep = TILE_SIZE / tan(RayAngle);
    xstep *= (IsRayFacingLeft && xstep > 0) ? -1 : 1;
    xstep *= (IsRayFacingRight && xstep < 0) ? -1 : 1;

    float nextHX = xintercept;
    float nextHY = yintercept;

    // Inceremnt xstep and ystep until we find a wall
    while (nextHX >= 0 && nextHX <= WINDOW_W && nextHY >= 0 && nextHY <= WINDOW_H) {
        float XCheck = nextHX;
        float YCheck = nextHY + (IsRayFacingUp ? -1 : 0);
        
        if (MapHasWallAt(XCheck, YCheck)) {
            // There is a wall hit.
            wallHitX_horizontal = nextHX;
            wallHitY_horizontal = nextHY;
            HorizontalWallContent = Map[(int)floor(YCheck / TILE_SIZE)][(int)floor(XCheck / TILE_SIZE)];
            foundHWallHit = TRUE;
            break;
        } else {
            nextHX += xstep;
            nextHY += ystep;
        }
    }
    


    // -----------------------------------
    // VERTICAL RAY-GRID INTERSECTION
    // -----------------------------------
    int foundVWallHit = FALSE;
    float wallHitX_vertical = 0;
    float wallHitY_vertical = 0;
    int VerticalWallContent = 0;

    // Find the x coordinate of the closest vertical grid intersection
    xintercept = floor(Player.X / TILE_SIZE) * TILE_SIZE;
    xintercept += IsRayFacingRight ? TILE_SIZE : 0; // instead of using conditional just * 1 or 0.
    
    yintercept = Player.Y + (xintercept - Player.X) * tan(RayAngle);

    // Calculate xstep and ystep (deltas)
    xstep = TILE_SIZE;
    xstep *= IsRayFacingLeft ? -1 : 1;

    ystep = TILE_SIZE * tan(RayAngle);
    ystep *= (IsRayFacingUp && ystep > 0) ? -1 : 1;
    ystep *= (IsRayFacingDown && ystep < 0) ? -1 : 1;

    float nextVX = xintercept;
    float nextVY = yintercept;

    // Inceremnt xstep and ystep until we find a wall
    while (nextVX >= 0 && nextVX <= WINDOW_W && nextVY >= 0 && nextVY <= WINDOW_H) {
        float XCheck = nextVX + (IsRayFacingLeft ? -1 : 0);
        float YCheck = nextVY;
        
        if (MapHasWallAt(XCheck, YCheck)) {
            // There is a wall hit.
            wallHitX_vertical = nextVX;
            wallHitY_vertical = nextVY;
            VerticalWallContent = Map[(int)floor(YCheck / TILE_SIZE)][(int)floor(XCheck / TILE_SIZE)];
            foundVWallHit = TRUE;
            break;
        } else {
            nextVX += xstep;
            nextVY += ystep;
        }
    }
    
    // Calculate both horizontal and vertical distances and choose
    // the smallest one.
    // TODO: Clear this shit. Looks like crap now.
    float HorizontalHitDistance = foundHWallHit
    ? DistanceBetweenPoints(Player.X, Player.Y, wallHitX_horizontal, wallHitY_horizontal)
    : FLT_MAX;
    
    float VerticalHitDistance = foundVWallHit
    ? DistanceBetweenPoints(Player.X, Player.Y, wallHitX_vertical, wallHitY_vertical)
    : FLT_MAX;
    
    // TODO: Save content of Ray into array of rays.
    if (VerticalHitDistance < HorizontalHitDistance) {
        Rays[StripId].Distance = VerticalHitDistance;
        Rays[StripId].WallHitX = wallHitX_vertical;
        Rays[StripId].WallHitY = wallHitY_vertical;
        Rays[StripId].WallHitContent = VerticalWallContent;
        Rays[StripId].WalHitVertical = TRUE;
    } else {
        Rays[StripId].Distance = HorizontalHitDistance;
        Rays[StripId].WallHitX = wallHitX_horizontal;
        Rays[StripId].WallHitY = wallHitY_horizontal;
        Rays[StripId].WallHitContent = HorizontalWallContent;
        Rays[StripId].WalHitVertical = FALSE;
    }
    
    Rays[StripId].RayAngle = RayAngle;
    Rays[StripId].IsRayFacingDown = IsRayFacingDown;
    Rays[StripId].IsRayFacingUp = IsRayFacingUp;
    Rays[StripId].IsRayFacingLeft = IsRayFacingLeft;
    Rays[StripId].IsRayFacingRight = IsRayFacingRight;
}


void CastAllRays(void) {
    float RayAngle = Player.RotationAngle - (FOV_ANGLE / 2);
    
    for (int StripId = 0; StripId < NUM_RAYS; ++StripId) {
        CastRay(RayAngle, StripId);
        
        RayAngle += FOV_ANGLE / NUM_RAYS;
    }
}


void RenderMap(void) {
    for (int i = 0; i < MAP_NUM_ROWS; i++) {
        for (int j = 0; j < MAP_NUM_COLS; j++) {
            int TileX = j * TILE_SIZE;
            int TileY = i * TILE_SIZE;
            int TileColor = Map[i][j] != 0 ? 255 : 0;

            SDL_SetRenderDrawColor(Renderer, TileColor, TileColor, TileColor, 255);

            SDL_Rect MapTileRect = {
                TileX * MINIMAP_SCALE_FACTOR,
                TileY * MINIMAP_SCALE_FACTOR,
                TILE_SIZE * MINIMAP_SCALE_FACTOR,
                TILE_SIZE * MINIMAP_SCALE_FACTOR
            };

            SDL_RenderFillRect(Renderer, &MapTileRect);
        }
    }
}


void RenderRays(void) {
    SDL_SetRenderDrawColor(Renderer, 255, 0, 0, 255);
    
    for (int i=0; i < NUM_RAYS; ++i) {
        SDL_RenderDrawLine(Renderer,
                           MINIMAP_SCALE_FACTOR * Player.X,
                           MINIMAP_SCALE_FACTOR * Player.Y,
                           MINIMAP_SCALE_FACTOR * Rays[i].WallHitX,
                           MINIMAP_SCALE_FACTOR * Rays[i].WallHitY);
    }
}


void ProcessInput(void) {
    SDL_Event Event;
    SDL_PollEvent(&Event);

    switch (Event.type) {
    case SDL_QUIT:
        IsGameRunning = FALSE;
        break;

    case SDL_KEYDOWN:
        if (Event.key.keysym.sym == SDLK_ESCAPE) {
            IsGameRunning = FALSE;
        }

        if (Event.key.keysym.sym == SDLK_UP) {
            Player.WalkDirection = +1;
        }

        if (Event.key.keysym.sym == SDLK_DOWN) {
            Player.WalkDirection = -1;
        }

        if (Event.key.keysym.sym == SDLK_LEFT) {
            Player.TurnDirection = -1;
        }

        if (Event.key.keysym.sym == SDLK_RIGHT) {
            Player.TurnDirection = +1;
        }

        break;

    case SDL_KEYUP:
        if (Event.key.keysym.sym == SDLK_UP) {
            Player.WalkDirection = 0;
        }

        if (Event.key.keysym.sym == SDLK_DOWN) {
            Player.WalkDirection = 0;
        }

        if (Event.key.keysym.sym == SDLK_LEFT) {
            Player.TurnDirection = 0;
        }

        if (Event.key.keysym.sym == SDLK_RIGHT) {
            Player.TurnDirection = 0;
        }

        break;

    }
}


void Update(void) {
    int TimeToWait = FRAME_TIME_LENGTH - (SDL_GetTicks() - TicksLastFrame);

    if (TimeToWait > 0 && TimeToWait <= FRAME_TIME_LENGTH) {
        SDL_Delay(TimeToWait);
    }

    float DeltaTime = (SDL_GetTicks() - TicksLastFrame) / 1000.0f;

    TicksLastFrame = SDL_GetTicks();

    // move player
    MovePlayer(DeltaTime);
    CastAllRays();
}

void Generate3DProjection(void) {
    for (int i=0; i < NUM_RAYS; i++) {
        float PerpendicularDistance = Rays[i].Distance * cos(Rays[i].RayAngle - Player.RotationAngle);
        
        // How far we are from projection plane.
        float DistanceToProjectionPlane = (WINDOW_W / 2) / tan(FOV_ANGLE / 2);
        
        float ProjectedWallHeight = (TILE_SIZE / PerpendicularDistance) * DistanceToProjectionPlane;
        
        // TODO: Replace, duplicate variable
        int WallStripHeight = ProjectedWallHeight;
        
        int WallTopPixel = (WINDOW_H / 2) - (WallStripHeight / 2);
        WallTopPixel = WallTopPixel < 0 ? 0 : WallTopPixel;
        
        int WallBottomPixel = (WINDOW_H / 2) + (WallStripHeight / 2);
        WallBottomPixel = WallBottomPixel > WINDOW_H ? WINDOW_H : WallBottomPixel;
        
        // Color for ceiling.
        for (int y = 0; y < WallTopPixel; ++y) {
            ColorBuffer[(WINDOW_W * y) + i] = 0xff333333;
        }
        
        // Render the wall from top to bottom pixels.
        for (int y = WallTopPixel; y < WallBottomPixel; ++y) {
            ColorBuffer[(WINDOW_W * y) + i] = Rays[i].WalHitVertical ? 0xffffffff : 0xffcccccc;
        }
        
        // Color for floor.
        for (int y = WallBottomPixel; y < WINDOW_H; ++y) {
            ColorBuffer[(WINDOW_W * y) + i] = 0xff777777;
        }
    }
}

void ClearColorBuffer(uint32_t Color) {
    for (int x = 0; x < WINDOW_W; ++x) {
        for (int y = 0; y < WINDOW_H; ++y) {
            ColorBuffer[(WINDOW_W * y) + x] = Color;
        }
    }
}


void RenderColorBuffer(void) {
    SDL_UpdateTexture(ColorBufferTexture, NULL, ColorBuffer, (int)((uint32_t)WINDOW_W * sizeof(uint32_t)));
    SDL_RenderCopy(Renderer, ColorBufferTexture, NULL, NULL);
}


void Render(void) {
    SDL_SetRenderDrawColor(Renderer, 0, 0, 0, 255);
    SDL_RenderClear(Renderer);
    
    Generate3DProjection();
    
    // Render scene to color buffer.
    RenderColorBuffer();

    // Clear the Color Buffer.
    ClearColorBuffer(0xff000000);
    
    // Render minimap.
    RenderMap();
    RenderRays();
    RenderPlayer();

    SDL_RenderPresent(Renderer);
}


int main(void) {
    IsGameRunning = InitializeWindow();

    Setup();

    while (IsGameRunning) {
        ProcessInput();
        Update();
        Render();
    }

    DestroyWindow();

    return 0;
}
```

And missing consts.h file with some, well, consts.

```
#ifndef CONSTS_H
#define CONSTS_H

#define FALSE 0
#define TRUE 1

#define PI 3.14159265
#define TWO_PI 6.28318530

#define TILE_SIZE 64
#define MAP_NUM_ROWS 13
#define MAP_NUM_COLS 20

#define MINIMAP_SCALE_FACTOR 0.25

#define WINDOW_W (MAP_NUM_COLS * TILE_SIZE)
#define WINDOW_H (MAP_NUM_ROWS * TILE_SIZE)

#define FOV_ANGLE (60 * (PI / 180))

#define NUM_RAYS WINDOW_W

#define FPS 60
#define FRAME_TIME_LENGTH (1000 / FPS)

#endif
```

This code is a mess. Don't use it. I assure you that I'll post better code when raycaster will looks like something useful. For now it's some crazy stuff in one place.

*Hinode Coder*
