# -practice-week5.md
5주차 과제
import pygame
import random
import sys

pygame.init()

def get_korean_font(size):
    candidates = ["malgungothic", "applegothic", "nanumgothic", "notosanscjk"]
    for name in candidates:
        font = pygame.font.SysFont(name, size)
        if font.get_ascent() > 0:
            return font
    return pygame.font.SysFont(None, size)

WIDTH, HEIGHT = 800, 600
FPS = 60

WHITE   = (255, 255, 255)
BLACK   = (0,   0,   0)
GRAY    = (20,  20,  40)
BLUE    = (50,  150, 255)
RED     = (220, 50,  50)
YELLOW  = (240, 220, 0)
GREEN   = (50,  220, 80)
ORANGE  = (240, 140, 0)
PURPLE  = (180, 50,  200) # 보스 색상 추가

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Space Shooter - Boss Update")
clock = pygame.time.Clock()
font = get_korean_font(36)
font_big = get_korean_font(72)

# --- 레벨 설정 ---
LEVELS = [
    {"enemy_speed": 2, "spawn": 60, "label": "Lv.1"},
    {"enemy_speed": 3, "spawn": 40, "label": "Lv.2"},
    {"enemy_speed": 5, "spawn": 25, "label": "Lv.3"},
]

PLAYER_W, PLAYER_H = 40, 40
ENEMY_W,  ENEMY_H  = 36, 36
FAST_W,   FAST_H   = 24, 30  # 빠른 몹 크기
BOSS_W,   BOSS_H   = 120, 80 # 보스 크기
BULLET_W, BULLET_H = 6,  14

def draw_player(surf, rect):
    cx = rect.centerx
    pygame.draw.polygon(surf, BLUE, [
        (cx, rect.top),
        (rect.left, rect.bottom),
        (cx, rect.bottom - 8),
        (rect.right, rect.bottom),
    ])
    pygame.draw.rect(surf, YELLOW, (cx - 4, rect.bottom - 10, 8, 10))

# 적 그리기 함수 업그레이드 (타입별로 다르게 그림)
def draw_enemy(surf, en):
    rect = en["rect"]
    cx = rect.centerx
    if en["type"] == "normal":
        pygame.draw.polygon(surf, RED, [
            (cx, rect.bottom), (rect.left, rect.top),
            (cx, rect.top + 8), (rect.right, rect.top)
        ])
    elif en["type"] == "fast":
        pygame.draw.polygon(surf, ORANGE, [
            (cx, rect.bottom), (rect.left, rect.top),
            (cx, rect.top + 5), (rect.right, rect.top)
        ])
    elif en["type"] == "boss":
        pygame.draw.ellipse(surf, PURPLE, rect)
        pygame.draw.rect(surf, GRAY, (rect.left + 10, rect.top + 20, 30, 10)) # 눈장식
        pygame.draw.rect(surf, GRAY, (rect.right - 40, rect.top + 20, 30, 10))
        
        # 보스 체력바
        hp_ratio = en["hp"] / en["max_hp"]
        pygame.draw.rect(surf, RED, (rect.left, rect.top - 15, rect.width, 8))
        pygame.draw.rect(surf, GREEN, (rect.left, rect.top - 15, int(rect.width * hp_ratio), 8))

# 적 스폰 함수 업그레이드 (딕셔너리 반환)
def spawn_enemy(level_cfg, enemy_type="normal"):
    if enemy_type == "normal":
        x = random.randint(0, WIDTH - ENEMY_W)
        return {"rect": pygame.Rect(x, -ENEMY_H, ENEMY_W, ENEMY_H), "type": "normal", "hp": 1, "sy": level_cfg["enemy_speed"], "sx": 0}
    elif enemy_type == "fast":
        x = random.randint(0, WIDTH - FAST_W)
        return {"rect": pygame.Rect(x, -FAST_H, FAST_W, FAST_H), "type": "fast", "hp": 1, "sy": level_cfg["enemy_speed"] * 1.8, "sx": 0}
    elif enemy_type == "boss":
        x = WIDTH // 2 - BOSS_W // 2
        # 보스는 체력이 높고 좌우로 움직입니다.
        return {"rect": pygame.Rect(x, 20, BOSS_W, BOSS_H), "type": "boss", "hp": 30, "max_hp": 30, "sy": 0, "sx": 3}

def draw_stars(stars):
    for s in stars:
        pygame.draw.circle(screen, WHITE, (s[0], s[1]), s[2])

def draw_hud(score, lives, level_cfg):
    screen.blit(font.render(f"Score: {score}", True, WHITE), (10, 10))
    screen.blit(font.render(f"Lives: {'♥ ' * lives}", True, RED), (WIDTH - 180, 10))
    screen.blit(font.render(level_cfg["label"], True, YELLOW), (WIDTH // 2 - 25, 10))

def game_over_screen(score):
    screen.fill((10, 10, 30))
    screen.blit(font_big.render("GAME OVER", True, RED), (220, 220))
    screen.blit(font.render(f"Score: {score}", True, WHITE), (350, 310))
    screen.blit(font.render("R: Restart   Q: Quit", True, WHITE), (270, 360))
    pygame.display.flip()
    while True:
        for e in pygame.event.get():
            if e.type == pygame.QUIT:
                return False
            if e.type == pygame.KEYDOWN:
                if e.key == pygame.K_r: return True
                if e.key == pygame.K_q: return False

def main():
    player = pygame.Rect(WIDTH // 2 - PLAYER_W // 2, HEIGHT - 70, PLAYER_W, PLAYER_H)
    bullets  = []
    enemies  = [] # 이제 사각형이 아닌 사전(Dict)들이 들어갑니다.
    score    = 0
    lives    = 3
    shoot_cd = 0
    spawn_timer = 0
    level_idx = 0
    level_cfg = LEVELS[level_idx]
    invincible = 0
    
    boss_active = False # 보스가 살아있는지 체크
    next_boss_score = 150 # 첫 보스 등장 점수

    stars = [(random.randint(0, WIDTH), random.randint(0, HEIGHT), random.randint(1, 2))
             for _ in range(80)]

    while True:
        clock.tick(FPS)

        for e in pygame.event.get():
            if e.type == pygame.QUIT:
                return False

        keys = pygame.key.get_pressed()
        if keys[pygame.K_LEFT]  and player.left  > 0:      player.x -= 6
        if keys[pygame.K_RIGHT] and player.right < WIDTH:   player.x += 6
        if keys[pygame.K_UP]    and player.top   > 0:      player.y -= 6
        if keys[pygame.K_DOWN]  and player.bottom < HEIGHT: player.y += 6

        shoot_cd -= 1
        if keys[pygame.K_SPACE] and shoot_cd <= 0:
            b = pygame.Rect(player.centerx - BULLET_W // 2, player.top, BULLET_W, BULLET_H)
            bullets.append(b)
            shoot_cd = 15

        bullets = [b for b in bullets if b.bottom > 0]
        for b in bullets:
            b.y -= 10

        # --- 적 스폰 로직 ---
        # 보스가 살아있을 땐 일반 적 스폰 속도를 늦춥니다 (원하지 않으면 삭제 가능)
        spawn_timer += 1
        spawn_limit = level_cfg["spawn"] * 2 if boss_active else level_cfg["spawn"]
        
        if spawn_timer >= spawn_limit:
            spawn_timer = 0
            # 20% 확률로 빠른 적 스폰
            if random.random() < 0.2:
                enemies.append(spawn_enemy(level_cfg, "fast"))
            else:
                enemies.append(spawn_enemy(level_cfg, "normal"))

        # --- 보스 등장 로직 ---
        if score >= next_boss_score and not boss_active:
            enemies.append(spawn_enemy(level_cfg, "boss"))
            boss_active = True
            next_boss_score += 200 # 다음 보스는 현재 요구점수 + 200점 후에 등장

        # --- 적 이동 로직 ---
        alive_enemies = []
        for en in enemies:
            en["rect"].y += en["sy"]
            en["rect"].x += en["sx"]
            
            # 보스 좌우 튕김 로직
            if en["type"] == "boss":
                if en["rect"].left <= 0 or en["rect"].right >= WIDTH:
                    en["sx"] *= -1 # 벽에 닿으면 방향 반전

            if en["rect"].top < HEIGHT:
                alive_enemies.append(en)
        enemies = alive_enemies

        # --- 총알 충돌 로직 ---
        hit_bullets = set()
        for bi, b in enumerate(bullets):
            for en in enemies:
                if b.colliderect(en["rect"]):
                    hit_bullets.add(bi)
                    en["hp"] -= 1 # 체력 감소
                    break # 총알 하나당 한 번만 타격
        
        # 체력이 0 이하인 적 처리 및 점수 획득
        surviving_enemies = []
        for en in enemies:
            if en["hp"] <= 0:
                if en["type"] == "boss":
                    score += 100
                    boss_active = False # 보스 처치
                elif en["type"] == "fast":
                    score += 20
                else:
                    score += 10
            else:
                surviving_enemies.append(en)
        enemies = surviving_enemies
        
        bullets = [b for i, b in enumerate(bullets) if i not in hit_bullets]

        level_idx = min(score // 50, len(LEVELS) - 1)
        level_cfg = LEVELS[level_idx]

        # --- 플레이어 피격 로직 ---
        if invincible > 0:
            invincible -= 1
        else:
            for en in enemies:
                if player.colliderect(en["rect"]):
                    lives -= 1
                    invincible = 90
                    
                    # 피격 시 화면에 있는 보스 외 일반 적들만 초기화
                    enemies = [e for e in enemies if e["type"] == "boss"]
                    
                    if lives <= 0:
                        return game_over_screen(score)
                    break

        # 배경 별 애니메이션 (간단한 눈속임용 아래로 흐르기)
        for i in range(len(stars)):
            stars[i] = (stars[i][0], (stars[i][1] + stars[i][2]) % HEIGHT, stars[i][2])

        screen.fill(GRAY)
        draw_stars(stars)

        for b in bullets:
            pygame.draw.rect(screen, YELLOW, b)

        for en in enemies:
            draw_enemy(screen, en)

        blink = (invincible // 10) % 2 == 0
        if blink:
            draw_player(screen, player)

        draw_hud(score, lives, level_cfg)
        pygame.display.flip()

# 무한 재귀호출을 막기 위해 바깥쪽에 루프를 씌웠습니다.
if __name__ == "__main__":
    while True:
        if not main():
            break
            항목 내 아이디어
게임 제목 : 박살
장르 : 슈팅
기반 템플릿 Space Shooter
핵심 규칙 목표: 5분 동안 살아남기      승리 조건: 5분 생존     패배 조건: 그 전에 사망
조작 방법 : 방향키 이동 스페이스 공격
차별점 기존 게임과 다른 점:                                                                                                                                         아래는 내가 만들 게임의 기획서 초안이야. 빈칸을 채우고 핵심 규칙을 더 구체적으로
만들어줘. 난이도가 어떻게 올라가는지도 제안해줘. 중간고사 3주 안에 완성 가능한
규모인지도 알려줘.
'박살'이라는 제목이 아주 직관적이고 타격감이 느껴져서 슈팅 게임과 정말 잘 어울립니다! 5분 생존이라는 명확한 목표도 플레이어에게 긴장감을 주기에 아주 좋은 설정이네요.

작성해주신 초안을 바탕으로 기획서를 구체화하고, 중간고사 과제로 손색이 없도록 차별점과 난이도 곡선을 제안해 드릴게요.

📝 게임 기획서: 박살 (Smash)
게임 제목: 박살

장르: 탑다운 2D 슈팅 (탄막 생존 기반)

기반 템플릿: Space Shooter

조작 방법: 방향키(이동), 스페이스바(공격/꾹 누르면 연사)

1. 핵심 규칙 (구체화)
승리 조건: 타이머가 5분 00초에 도달할 때까지 플레이어 기체 생존.

패배 조건: 5분을 채우지 못하고 플레이어의 체력(HP)이 0이 되면 즉시 게임 오버.

체력 시스템: 목숨(Life) 3개 또는 체력 게이지(3번 피격 시 사망)로 설정.

점수(Score) 시스템: 생존이 주 목표지만, 적을 파괴할 때마다 점수가 오릅니다. 5분 생존 시 최종 점수로 랭킹을 매겨 반복 플레이 동기를 부여합니다.

2. 차별점 (기존 스페이스 슈터와의 차이)
제목인 '박살'의 느낌을 살리기 위해 **[파편(Shrapnel) 연쇄 시스템]**을 제안합니다.

파편 연쇄 폭발: 적을 파괴하면 단순히 사라지는 것이 아니라, 사방으로 '파편'이 튀어 나갑니다. 이 파편이 다른 적에게 맞으면 데미지를 주어 **연쇄 폭발(Chain Reaction)**을 일으킵니다.

시각적 쾌감: 위기에 몰렸을 때 적 한 마리를 터뜨려 화면 전체의 적이 와장창 '박살' 나는 시각적/청각적 쾌감을 극대화합니다.

위험과 보상: 파편 중 일부는 플레이어의 무기를 강화해 주는 '강화 조각'으로 떨어집니다. 적진으로 위험을 무릅쓰고 들어가 조각을 먹으면 무기가 점점 강해집니다(단발 -> 3갈래 -> 5갈래 등).

3. 난이도 상승 곡선 (5분 생존 페이즈)
단순히 적이 많아지는 것보다, 시간대별로 뚜렷한 페이즈(Phase)를 나누는 것이 지루하지 않습니다.

0분 ~ 1분 (탐색전): 느리게 일직선으로 다가오는 기본 적만 등장. 플레이어가 조작과 '연쇄 폭발'의 감을 익히는 시간.

1분 ~ 2분 (속도전): 체력은 약하지만 이동 속도가 2배 빠른 '돌격형 적'이 섞여 나옴. 플레이어의 빠른 회피 피지컬 요구.

2분 ~ 3분 (물량전): 화면 가장자리에서 수많은 적이 쏟아져 나옴. 이때 무기 업그레이드와 연쇄 폭발을 적극적으로 활용해야만 화면을 정리할 수 있음.

3분 ~ 4분 (탱커 등장): 크기가 크고 체력이 5배 높은 '중형 적(운석 등)' 등장. 이 적을 부수면 작은 적 3~4마리로 '분열'함.

4분 ~ 5분 (클라이맥스): 모든 종류의 적이 총공세를 펼침. 마지막 1분은 탄막을 피하며 살아남는 극한의 서바이벌.

⏳ 3주 안에 완성 가능한 규모일까?
결론부터 말씀드리면, "충분히 가능하며 아주 적절한 규모"입니다. 슈팅 게임은 파이썬(Pygame)이나 유니티(Unity) 같은 엔진에서 가장 구현하기 쉬운 장르 중 하나입니다. 중력을 계산할 필요가 없고, 충돌 판정과 타이머만 있으면 되기 때문입니다.

추천 개발 일정:

1주차 (핵심 기능): 플레이어 이동, 총알 발사, 기본 적 생성, 충돌 판정(죽음), 5분 타이머 및 UI 구현. (여기까지만 해도 뼈대는 완성됩니다.)

2주차 (차별점 & 레벨 디자인): '파편 시스템(적 사망 시 파편 총알 생성)' 구현, 시간에 따라 적 스폰 속도와 종류가 바뀌는 페이즈 시스템 적용.

3주차 (폴리싱): 타격감 추가(폭발음, 이펙트, 화면 흔들림 효과 등), 버그 수정, 난이도 밸런스 조절.

특히 3주차의 '화면 흔들림'이나 '파괴음' 같은 효과만 잘 넣어도 교수님과 친구들에게 퀄리티가 높다는 인상을 줄 수 있습니다.

이 기획에서 적을 터뜨렸을 때 나오는 '파편'을 무기 업그레이드 아이템으로 쓰게 만드는 것이 마음에 드시나요, 아니면 파편 자체가 다른 적을 연쇄적으로 부수는 폭탄 같은 역할이 되는 것이 더 끌리시나요?
    pygame.quit()
    sys.exit()
내가 기획한 게임에서 [차별점] 기능을 구현하려면 Pygame 코드에서 어떤 부분을
어떻게 수정해야 해? 수정이 어려운 부분이 있으면 미리 알려줘.
'박살'의 핵심 재미 요소인 **[파편 연쇄 폭발]**과 **[무기 강화 조각]** 시스템을 현재 코드에 넣으려면 꽤 흥미로운 작업들이 필요합니다. 

기존 코드가 딕셔너리(`{}`) 형태로 적을 관리하도록 업그레이드되었기 때문에, 파편 시스템을 추가하는 것 자체는 어렵지 않습니다. 하지만 몇 가지 주의해야 할 점도 함께 존재합니다. 어떻게 수정해야 하는지, 그리고 어떤 부분이 까다로울 수 있는지 짚어드릴게요.

---

### 🛠️ 구현을 위한 핵심 수정 포인트 4가지

#### 1. 수학 모듈 추가 및 파편/아이템 리스트 생성
파편이 사방(360도)으로 퍼져나가게 하려면 삼각함수(sin, cos)가 필요합니다. 또한 파편과 아이템을 관리할 빈 리스트와 무기 레벨 변수가 필요합니다.
```python
import math # 맨 위에 추가

# main() 함수 내부 변수 초기화 부분
shrapnels = [] # 적을 공격하는 파편
items = []     # 플레이어 무기 강화 조각
weapon_level = 1 # 무기 레벨 (최대 3~5)
```

#### 2. 파편 생성 함수 만들기 (`main` 밖)
적이 죽은 위치(x, y)를 받아서 여러 개의 파편을 흩뿌리는 함수를 만들어야 합니다.
```python
def spawn_shrapnels(x, y, count, is_boss=False):
    new_shrapnels = []
    new_items = []
    
    for i in range(count):
        angle = random.uniform(0, math.pi * 2) # 0~360도 랜덤 방향
        speed = random.uniform(3, 7)
        vx = math.cos(angle) * speed
        vy = math.sin(angle) * speed
        
        # 10% 확률로 공격용 파편 대신 강화 아이템 생성
        if random.random() < 0.1:
            new_items.append({"rect": pygame.Rect(x, y, 10, 10), "vx": vx, "vy": vy})
        else:
            new_shrapnels.append({"rect": pygame.Rect(x, y, 6, 6), "vx": vx, "vy": vy, "life": 60}) # life는 파편 생존 시간(프레임)
            
    return new_shrapnels, new_items
```

#### 3. 적 사망 로직 수정 (연쇄 폭발의 시작)
기존에 총알에 맞아 적의 `hp`가 0이 되면 점수만 올리고 끝났지만, 이제 여기서 방금 만든 `spawn_shrapnels` 함수를 호출해야 합니다.
```python
# 체력이 0 이하인 적 처리 부분 수정
surviving_enemies = []
for en in enemies:
    if en["hp"] <= 0:
        # 적 종류에 따라 파편 갯수 다르게 설정
        drop_count = 20 if en["type"] == "boss" else 5
        
        # 파편과 아이템 생성 후 리스트에 추가
        new_s, new_i = spawn_shrapnels(en["rect"].centerx, en["rect"].centery, drop_count)
        shrapnels.extend(new_s)
        items.extend(new_i)
        
        # (기존 점수 획득 로직 유지)
    else:
        surviving_enemies.append(en)
enemies = surviving_enemies
```

#### 4. 파편 이동 및 연쇄 충돌 판정 (가장 중요)
메인 루프 안에서 파편들을 이동시키고, **파편이 다른 적과 부딪혔는지** 확인하는 코드가 필요합니다.
```python
# 파편 이동 및 적 충돌 로직
alive_shrapnels = []
for s in shrapnels:
    s["rect"].x += s["vx"]
    s["rect"].y += s["vy"]
    s["life"] -= 1
    
    hit_something = False
    for en in enemies:
        if s["rect"].colliderect(en["rect"]):
            en["hp"] -= 1 # 파편에 맞아도 체력 감소! (연쇄 폭발)
            hit_something = True
            break
            
    if not hit_something and s["life"] > 0 and 0 < s["rect"].x < WIDTH and 0 < s["rect"].y < HEIGHT:
        alive_shrapnels.append(s)
shrapnels = alive_shrapnels
```

---

### ⚠️ 미리 알아두어야 할 "어려운 점 (리스크)"

1. **프레임 드랍 (최적화 문제)**
   * **원인:** 화면에 적이 10마리가 있는데, 연쇄 폭발이 일어나 파편이 50개 생성되었다고 가정해 봅시다. 매 프레임마다 파편 50개와 적 10마리의 충돌을 일일이 계산해야 합니다. 운이 나빠 화면 전체가 폭발하면 파편이 수백 개로 늘어나 컴퓨터가 버벅거릴(렉이 걸릴) 수 있습니다.
   * **해결책:** 파편의 개수를 적당히 제한하거나, 파편의 생존 시간(`life`)을 짧게 주어 화면 밖으로 나가기 전에 스스로 소멸하게 만들어야 합니다.
2. **무기 레벨업 시스템 (다중 발사)**
   * **원인:** 아이템을 먹고 `weapon_level`이 올랐을 때, 스페이스바를 누르면 총알이 일직선으로 1발 나가는 게 아니라 부채꼴 모양으로 3발, 5발씩 나가게 수학(각도) 계산을 넣어야 합니다. 이 부분의 코드가 조금 복잡해질 수 있습니다.

파편 연쇄 폭발을 우선적으로 구현해 볼까요, 아니면 플레이어의 무기가 다방향으로 나가는 다중 발사 기능을 먼저 구현해 볼까요?
