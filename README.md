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
    pygame.quit()
    sys.exit()
