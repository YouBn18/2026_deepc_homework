# 탄막 피하기

## 소스코드
```
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <time.h>
#include <math.h>
#include "raylib.h"

#define SCREEN_WIDTH 800
#define SCREEN_HEIGHT 600
#define MAX_BULLETS 2000

// 게임 상태 관리
typedef enum {
   GAME_STATE_MENU = 0,
   GAME_STATE_COUNTDOWN,
   GAME_STATE_PLAYING,
   GAME_STATE_GAMEOVER
} GameState;

// 난이도 모드
typedef enum {
   MODE_NORMAL = 0,
   MODE_HARD
} GameMode;

// 탄막 구조체
typedef struct {
   float x, y;
   float vx, vy;
   float radius;
   bool active;
} Bullet;

// 아이템 유형 및 구조체
typedef enum {
   ITEM_SHIELD = 0,
   ITEM_BOMB,
   ITEM_TENT,
   ITEM_HEART,
   ITEM_MAX
} ItemType;

typedef struct {
   float x, y;
   float size;
   ItemType type;
   bool active;
} Item;

// 고정형 텐트 구조체
typedef struct {
   float x, y;
   float radius;
   float timer;
   bool active;
} Tent;

Bullet bullets[MAX_BULLETS];
Item activeItem = { 0 };
Tent playerTent = { 0 };

void InitBullets(void) {
   for (int i = 0; i < MAX_BULLETS; i++) {
      bullets[i].active = false;
   }
}

// 맵 바깥 사방에서 탄막 생성
void SpawnBullet(void) {
   int index = -1;
   for (int i = 0; i < MAX_BULLETS; i++) {
      if (!bullets[i].active) {
         index = i;
         break;
      }
   }
   if (index == -1) return;

   float radius = 5.0f;
   float x, y, targetX, targetY;
   int side = rand() % 4;

   switch (side) {
   case 0:
      x = (float)(rand() % SCREEN_WIDTH);
      y = -radius;
      targetX = (float)(rand() % SCREEN_WIDTH);
      targetY = SCREEN_HEIGHT + radius;
      break;
   case 1:
      x = (float)(rand() % SCREEN_WIDTH);
      y = SCREEN_HEIGHT + radius;
      targetX = (float)(rand() % SCREEN_WIDTH);
      targetY = -radius;
      break;
   case 2:
      x = -radius;
      y = (float)(rand() % SCREEN_HEIGHT);
      targetX = SCREEN_WIDTH + radius;
      targetY = (float)(rand() % SCREEN_HEIGHT);
      break;
   default:
      x = SCREEN_WIDTH + radius;
      y = (float)(rand() % SCREEN_HEIGHT);
      targetX = -radius;
      targetY = (float)(rand() % SCREEN_HEIGHT);
      break;
   }

   float angle = atan2f(targetY - y, targetX - x);
   float speed = 200.0f + (rand() % 100);

   bullets[index].x = x;
   bullets[index].y = y;
   bullets[index].vx = cosf(angle) * speed;
   bullets[index].vy = sinf(angle) * speed;
   bullets[index].radius = radius;
   bullets[index].active = true;
}

int main(void) {
   InitWindow(SCREEN_WIDTH, SCREEN_HEIGHT, "Raylib Bullet Hell - Balanced Nerf Edition");
   SetTargetFPS(60);

   srand((unsigned int)time(NULL));
   InitBullets();

   GameState gameState = GAME_STATE_MENU;
   GameMode gameMode = MODE_NORMAL;

   // 게임 플레이 변수
   float playerRadius = 5.0f * 1.2f;
   int lives = 3;
   long long score = 0;

   float timeTimer = 0.0f;
   float spawnTimer = 0.0f;
   float elapsedTime = 0.0f;

   // 카운트다운 관련 변수 (0.5초 간격)
   float countdownTimer = 0.5f;
   int countdownStep = 3;

   // 무적 및 피격 깜빡임
   float invulnerableTimer = 0.0f;
   bool isInvulnerable = false;
   float damageBlinkTimer = 0.0f;

   float itemSpawnTimer = 0.0f;
   float shieldTimer = 0.0f;
   bool hasShield = false;

   while (!WindowShouldClose()) {
      float dt = GetFrameTime();
      Vector2 mousePos = GetMousePosition();
      bool mousePressed = IsMouseButtonPressed(MOUSE_BUTTON_LEFT);

      // 1. 메인 메뉴 상태
      if (gameState == GAME_STATE_MENU) {
         ShowCursor();

         Rectangle normalButton = { SCREEN_WIDTH / 2 - 150, 240, 300, 50 };
         Rectangle hardButton = { SCREEN_WIDTH / 2 - 150, 320, 300, 50 };

         if (CheckCollisionPointRec(mousePos, normalButton) && mousePressed) {
            gameMode = MODE_NORMAL;
            InitBullets();
            lives = 3;
            score = 0;
            timeTimer = 0.0f;
            spawnTimer = 0.0f;
            elapsedTime = 0.0f;
            invulnerableTimer = 0.0f;
            isInvulnerable = false;
            itemSpawnTimer = 0.0f;
            hasShield = false;
            playerTent.active = false;
            activeItem.active = false;

            countdownStep = 3;
            countdownTimer = 0.5f;
            gameState = GAME_STATE_COUNTDOWN;
            HideCursor();
         }

         if (CheckCollisionPointRec(mousePos, hardButton) && mousePressed) {
            gameMode = MODE_HARD;
            InitBullets();
            lives = 3;
            score = 0;
            timeTimer = 0.0f;
            spawnTimer = 0.0f;
            elapsedTime = 0.0f;
            invulnerableTimer = 0.0f;
            isInvulnerable = false;
            itemSpawnTimer = 0.0f;
            hasShield = false;
            playerTent.active = false;
            activeItem.active = false;

            countdownStep = 3;
            countdownTimer = 0.5f;
            gameState = GAME_STATE_COUNTDOWN;
            HideCursor();
         }
      }
      // 2. 카운트다운 상태 (3 -> 2 -> 1 -> START)
      else if (gameState == GAME_STATE_COUNTDOWN) {
         countdownTimer -= dt;
         if (countdownTimer <= 0.0f) {
            countdownStep--;
            if (countdownStep >= 0) {
               countdownTimer = 0.5f;
            }
            else {
               gameState = GAME_STATE_PLAYING;
            }
         }
      }
      // 3. 게임 플레이 중 상태
      else if (gameState == GAME_STATE_PLAYING) {
         elapsedTime += dt;

         if (isInvulnerable) {
            invulnerableTimer -= dt;
            if (invulnerableTimer <= 0) isInvulnerable = false;
         }

         if (damageBlinkTimer > 0) {
            damageBlinkTimer -= dt;
         }

         if (hasShield) {
            shieldTimer -= dt;
            if (shieldTimer <= 0) hasShield = false;
         }

         if (playerTent.active) {
            playerTent.timer -= dt;
            if (playerTent.timer <= 0) playerTent.active = false;
         }

         timeTimer += dt;
         if (timeTimer >= 0.1f) {
            score += 10;
            timeTimer -= 0.1f;
         }

         // [새로운 탄막 생성 밸런스 공식 적용]
         float spawnRate = 2.0f;
         float minutes = elapsedTime / 60.0f;

         if (gameMode == MODE_NORMAL) {
            // Normal 모드 밸런스
            if (elapsedTime <= 60.0f) {
               spawnRate = 2.0f + (elapsedTime / 60.0f) * 4.0f; // 0분: 2개 -> 1분: 6개
            }
            else if (elapsedTime <= 120.0f) {
               spawnRate = 6.0f + ((elapsedTime - 60.0f) / 60.0f) * 4.0f; // 1분: 6개 -> 2분: 10개
            }
            else if (elapsedTime <= 180.0f) {
               spawnRate = 10.0f + ((elapsedTime - 120.0f) / 60.0f) * 3.0f; // 2분: 10개 -> 3분: 13개
            }
            else if (elapsedTime <= 240.0f) {
               spawnRate = 13.0f + ((elapsedTime - 180.0f) / 60.0f) * 3.0f; // 3분: 13개 -> 4분: 16개
            }
            else if (elapsedTime <= 300.0f) {
               spawnRate = 16.0f + ((elapsedTime - 240.0f) / 60.0f) * 2.0f; // 4분: 16개 -> 5분: 18개
            }
            else if (elapsedTime <= 360.0f) {
               spawnRate = 18.0f + ((elapsedTime - 300.0f) / 60.0f) * 2.0f; // 5분: 18개 -> 6분: 20개
            }
            else {
               // 6분 이후: 25개가 될 때까지 분당 1개씩 추가 생성 (최대 25개 한계)
               spawnRate = 20.0f + (minutes - 6.0f) * 1.0f;
               if (spawnRate > 25.0f) spawnRate = 25.0f;
            }
         }
         else {
            // Hard 모드 밸런스
            if (elapsedTime <= 60.0f) {
               spawnRate = 3.0f + (elapsedTime / 60.0f) * 7.0f; // 0분: 3개 -> 1분: 10개
            }
            else if (elapsedTime <= 120.0f) {
               spawnRate = 10.0f + ((elapsedTime - 60.0f) / 60.0f) * 6.0f; // 1분: 10개 -> 2분: 16개
            }
            else if (elapsedTime <= 180.0f) {
               spawnRate = 16.0f + ((elapsedTime - 120.0f) / 60.0f) * 5.0f; // 2분: 16개 -> 3분: 21개
            }
            else if (elapsedTime <= 240.0f) {
               spawnRate = 21.0f + ((elapsedTime - 180.0f) / 60.0f) * 4.0f; // 3분: 21개 -> 4분: 25개
            }
            else if (elapsedTime <= 300.0f) {
               spawnRate = 25.0f + ((elapsedTime - 240.0f) / 60.0f) * 3.0f; // 4분: 25개 -> 5분: 28개
            }
            else if (elapsedTime <= 360.0f) {
               spawnRate = 28.0f + ((elapsedTime - 300.0f) / 60.0f) * 2.0f; // 5분: 28개 -> 6분: 30개
            }
            else {
               // 6분 이후: 40개가 될 때까지 분당 1개씩 추가 생성 (최대 40개 한계)
               spawnRate = 30.0f + (minutes - 6.0f) * 1.0f;
               if (spawnRate > 40.0f) spawnRate = 40.0f;
            }
         }

         float spawnInterval = 1.0f / spawnRate;
         spawnTimer += dt;
         while (spawnTimer >= spawnInterval) {
            SpawnBullet();
            spawnTimer -= spawnInterval;
         }

         // 아이템 스폰 (15초 주기, 미수거 시 자동 소멸 후 재생성)
         itemSpawnTimer += dt;
         if (itemSpawnTimer >= 15.0f) {
            itemSpawnTimer = 0.0f;
            ItemType chosenType;
            while (true) {
               chosenType = (ItemType)(rand() % ITEM_MAX);
               if (chosenType == ITEM_HEART && lives >= 3) continue;
               break;
            }
            activeItem.x = 200.0f + (rand() % 400);
            activeItem.y = 150.0f + (rand() % 300);
            activeItem.size = playerRadius * 1.25f * 2.0f;
            activeItem.type = chosenType;
            activeItem.active = true;
         }

         // 아이템 획득 충돌
         if (activeItem.active) {
            if (CheckCollisionCircles(mousePos, playerRadius, (Vector2) { activeItem.x, activeItem.y }, activeItem.size / 2.0f)) {
               if (activeItem.type == ITEM_SHIELD) {
                  hasShield = true;
                  shieldTimer = 10.0f;
               }
               else if (activeItem.type == ITEM_BOMB) {
                  for (int i = 0; i < MAX_BULLETS; i++) {
                     if (bullets[i].active) {
                        float dx = bullets[i].x - mousePos.x;
                        float dy = bullets[i].y - mousePos.y;
                        float dist = sqrtf(dx * dx + dy * dy);
                        if (dist < 1.0f) dist = 1.0f;
                        float pushSpeed = 350.0f;
                        bullets[i].vx = (dx / dist) * pushSpeed;
                        bullets[i].vy = (dy / dist) * pushSpeed;
                     }
                  }
               }
               else if (activeItem.type == ITEM_TENT) {
                  playerTent.x = mousePos.x;
                  playerTent.y = mousePos.y;
                  playerTent.radius = playerRadius * 2.5f * 1.5f;
                  playerTent.timer = 5.0f;
                  playerTent.active = true;
               }
               else if (activeItem.type == ITEM_HEART) {
                  if (lives < 3) lives++;
               }
               activeItem.active = false;
            }
         }

         // 탄막 이동 및 충돌
         for (int i = 0; i < MAX_BULLETS; i++) {
            if (!bullets[i].active) continue;

            if (playerTent.active) {
               if (CheckCollisionCircles((Vector2) { playerTent.x, playerTent.y }, playerTent.radius, (Vector2) { bullets[i].x, bullets[i].y }, bullets[i].radius)) {
                  bullets[i].active = false;
                  continue;
               }
            }

            bullets[i].x += bullets[i].vx * dt;
            bullets[i].y += bullets[i].vy * dt;

            if (bullets[i].x < -20 || bullets[i].x > SCREEN_WIDTH + 20 ||
               bullets[i].y < -20 || bullets[i].y > SCREEN_HEIGHT + 20) {
               bullets[i].active = false;
               score += 25;
            }

            if (!isInvulnerable && CheckCollisionCircles(mousePos, playerRadius, (Vector2) { bullets[i].x, bullets[i].y }, bullets[i].radius)) {
               if (hasShield) {
                  hasShield = false;
                  isInvulnerable = true;
                  invulnerableTimer = 2.0f;
                  bullets[i].active = false;
               }
               else {
                  lives--;
                  damageBlinkTimer = 1.0f;
                  bullets[i].active = false;
                  if (lives <= 0) {
                     gameState = GAME_STATE_GAMEOVER;
                  }
                  else {
                     isInvulnerable = true;
                     invulnerableTimer = 2.0f;
                  }
               }
            }
         }
      }
      // 4. 게임 오버 상태
      else if (gameState == GAME_STATE_GAMEOVER) {
         ShowCursor();
         if (IsKeyPressed(KEY_R)) {
            gameState = GAME_STATE_MENU;
         }
      }

      // --- 렌더링 ---
      BeginDrawing();
      ClearBackground((Color) { 15, 15, 20, 255 });

      if (gameState == GAME_STATE_MENU) {
         DrawText("BULLET HELL DODGE", SCREEN_WIDTH / 2 - 220, 120, 36, YELLOW);
         DrawText("Click a Button to Start", SCREEN_WIDTH / 2 - 110, 180, 18, LIGHTGRAY);

         Rectangle normalButton = { SCREEN_WIDTH / 2 - 150, 240, 300, 50 };
         Rectangle hardButton = { SCREEN_WIDTH / 2 - 150, 320, 300, 50 };

         bool hoverNormal = CheckCollisionPointRec(mousePos, normalButton);
         bool hoverHard = CheckCollisionPointRec(mousePos, hardButton);

         DrawRectangleRec(normalButton, hoverNormal ? (Color) { 60, 160, 80, 255 } : (Color) { 40, 120, 60, 255 });
         DrawRectangleLinesEx(normalButton, 2, GREEN);
         DrawText("NORMAL MODE", SCREEN_WIDTH / 2 - 75, 255, 18, RAYWHITE);

         DrawRectangleRec(hardButton, hoverHard ? (Color) { 180, 60, 60, 255 } : (Color) { 140, 40, 40, 255 });
         DrawRectangleLinesEx(hardButton, 2, RED);
         DrawText("HARD MODE", SCREEN_WIDTH / 2 - 60, 335, 18, RAYWHITE);

         DrawText("Control: Mouse Cursor | Avoid Bullets & Grab Items!", SCREEN_WIDTH / 2 - 200, 450, 14, GRAY);
      }
      else {
         for (int i = 0; i < MAX_BULLETS; i++) {
            if (bullets[i].active) {
               DrawCircleV((Vector2) { bullets[i].x, bullets[i].y }, bullets[i].radius, RED);
            }
         }

         if (activeItem.active) {
            Color itemColor = WHITE;
            if (activeItem.type == ITEM_SHIELD) itemColor = BLUE;
            else if (activeItem.type == ITEM_BOMB) itemColor = ORANGE;
            else if (activeItem.type == ITEM_TENT) itemColor = BROWN;
            else if (activeItem.type == ITEM_HEART) itemColor = MAGENTA;

            DrawCircle((int)activeItem.x, (int)activeItem.y, activeItem.size / 2.0f, itemColor);
            DrawCircleLines((int)activeItem.x, (int)activeItem.y, activeItem.size / 2.0f, WHITE);
         }

         if (playerTent.active) {
            bool tentBlink = (playerTent.timer > 2.0f) || ((int)(GetTime() * 10) % 2 == 0);
            if (tentBlink) {
               DrawCircleLines((int)playerTent.x, (int)playerTent.y, playerTent.radius, BROWN);
               DrawCircle((int)playerTent.x, (int)playerTent.y, playerTent.radius, (Color) { 139, 69, 19, 40 });
            }
         }

         if (gameState == GAME_STATE_PLAYING) {
            bool drawPlayer = true;
            if (isInvulnerable) {
               if ((int)(invulnerableTimer * 10) % 2 == 0) drawPlayer = false;
            }

            if (drawPlayer) {
               DrawCircleV(mousePos, playerRadius, LIME);
               DrawCircleLines((int)mousePos.x, (int)mousePos.y, playerRadius + 2.0f, WHITE);
            }

            if (hasShield) {
               bool shieldBlink = (shieldTimer > 2.0f) || ((int)(GetTime() * 10) % 2 == 0);
               if (shieldBlink) {
                  DrawCircleLines((int)mousePos.x, (int)mousePos.y, playerRadius + 6.0f, BLUE);
               }
            }
         }

         // 상단 UI 정보
         DrawText(TextFormat("SCORE: %lld", score), 20, 20, 20, YELLOW);
         DrawText(TextFormat("MODE: %s", gameMode == MODE_NORMAL ? "NORMAL" : "HARD"), SCREEN_WIDTH - 150, 20, 18, gameMode == MODE_NORMAL ? GREEN : RED);
         DrawText(TextFormat("TIME: %.1fs", elapsedTime), 20, 80, 16, RAYWHITE);

         // 원형 목숨 UI
         DrawText("LIVES:", 20, 50, 20, LIME);
         for (int i = 0; i < 3; i++) {
            Color circleColor = LIME;
            if (i >= lives) {
               circleColor = (Color){ 60, 60, 60, 255 };
            }
            else {
               if (damageBlinkTimer > 0 && (int)(damageBlinkTimer * 20) % 2 == 0) {
                  circleColor = WHITE;
               }
            }
            DrawCircle(95 + (i * 25), 60, 8.0f, circleColor);
            DrawCircleLines(95 + (i * 25), 60, 8.0f, WHITE);
         }

         if (hasShield) DrawText(TextFormat("SHIELD: %.1fs", shieldTimer), 20, 110, 16, BLUE);
         if (playerTent.active) DrawText(TextFormat("TENT ZONE: %.1fs", playerTent.timer), 20, 130, 16, BROWN);

         // 카운트다운 오버레이
         if (gameState == GAME_STATE_COUNTDOWN) {
            DrawRectangle(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT, (Color) { 0, 0, 0, 160 });

            if (countdownStep > 0) {
               DrawText(TextFormat("%d", countdownStep), SCREEN_WIDTH / 2 - 20, SCREEN_HEIGHT / 2 - 40, 60, YELLOW);
            }
            else {
               DrawText("S T A R T !", SCREEN_WIDTH / 2 - 110, SCREEN_HEIGHT / 2 - 30, 50, GREEN);
            }
         }

         if (gameState == GAME_STATE_GAMEOVER) {
            DrawText("G A M E   O V E R", SCREEN_WIDTH / 2 - 140, SCREEN_HEIGHT / 2 - 40, 30, RED);
            DrawText(TextFormat("FINAL SCORE: %lld", score), SCREEN_WIDTH / 2 - 110, SCREEN_HEIGHT / 2 + 10, 20, YELLOW);
            DrawText("Press 'R' to Return to Menu", SCREEN_WIDTH / 2 - 125, SCREEN_HEIGHT / 2 + 45, 16, GRAY);
         }
      }

      EndDrawing();
   }

   CloseWindow();
   return 0;
}
```
## 설명
- 실행을 하면 두 가지 모드를 선택할 수 있다. normal mode 와 hard mode 가 존재하고, hard mode는 normal mode보다 더 많은 탄막을 생성한다.
- 캐릭터는 마우스 커서를 이동해서 움직일 수 있다.
- life 기능이 있다. 3개가 주어지고 탄막에 닿일 때마다 life가 1씩 줄어든다.
- 피격 시 무적 시간이 있고, 2초간 지속된다.
- 점수 계산 방식은 2가지로 설정하였다. 첫 번째는 시간당(0.1초마다 10점), 두 번째는 탄막 회피 성공마다(탄막이 생성된 벽에서 다른 벽으로 완전히 이동될 때 개수마다 25점)
- 아이템 기능도 구현해보았다. 아이템은 4가지로 구성된다. 아이템은 15초마다 생성되고, 누적되지 않는다.
  1. 쉴드: 아이템을 얻은 이후로 탄막의 공격을 1회 막아준다. 쉴드 지속시간은 10초이고, 지속시간이 끝나거나 탄막에 맞을 시 소멸된다.
  2. 탄막풍: 아이템을 얻은 즉시 필드에 있는 모든 탄이 캐릭터로부터 멀어지는 방향으로 날아간다.
  3. 텐트: 아이템을 얻은 즉시 그 자리에 안전 구역이 생긴다. 텐트 지속시간은 5초이고, 지속시간이 끝나면 텐트 효과가 사라진다.
  4. 생명: 아이템을 얻은 즉시 life가 1 늘어난다. 단, life가 2 이하일 때만 생성하는 아이템으로 life가 3이라면 다른 아이템을 생성한다.

## 소감
이번에 게임을 만들어보면서 AI의 기술이 굉장히 발전했다는 것을 알게 되었다. AI를 이용해서 게임을 만들 때 원하는 기능을 말로 설명만 해주면 코드를 생성하는 것을 보고 개발을 하기 훨씬 편해졌다는 것을 느꼈다.
이번 탄막 피하기 게임을 만들 때 단순히 탄막만 피하는 게임으로 두기는 아쉬워서 다양한 게임의 특성을 추가해보았다. 이 게임을 추후에는 랭킹 시스템으로 데이터베이스와 연결하여 순위를 나열할 수 있게도 만들 수 있을 것 같다.
좋은 과제를 만들어주셔서 감사합니다 교수님. 항상 건강하시길 바랍니다.
