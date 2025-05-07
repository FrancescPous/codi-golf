import pygame
import math
import random
import time

# Configuració de pantalla
WIDTH, HEIGHT = 1280, 720
WHITE = (255, 255, 255)
GREEN = (0, 128, 0)
RED = (255, 0, 0)
BALL_COLOR = (255, 255, 0)
BLUE = (0, 0, 255)
BLACK = (0, 0, 0)

pygame.init()
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Golf Game")
clock = pygame.time.Clock()
font = pygame.font.Font(None, 36)

# Carregar sons
hit_sound = pygame.mixer.Sound("img/mixkit-impact-of-a-blow-2150.wav")

# Estat del joc
game_state = "menu"
start_time = 0
angle = 0
force = 0
max_force = 10
shooting = False
show_congrats = False
congrats_time = 0

# Dades per a estadístiques i controls post-nivell
shots_count = 0
level_start_time = 0
level_end_time = 0
post_level_state = False  # Estat per mostrar pantalla post nivell
post_level_choice = None  # 'repeat' o 'next'

# Carregar imatges
grass_texture = pygame.image.load("img/cesped.jpg")  # Imatge de gespa
grass_texture = pygame.transform.smoothscale(grass_texture, (WIDTH, HEIGHT))

# Punt d'inici global de la pilota per reposicionar en cas de tocar obstacle
ball_start_pos = (100, 500)


# Funció per dibuixar text a la pantalla
def draw_text(text, x, y, color=WHITE, center=False):
    img = font.render(text, True, color)
    if center:
        rect = img.get_rect(center=(x, y))
        screen.blit(img, rect)
    else:
        screen.blit(img, (x, y))


# Classe per a la pilota
class Ball:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.radius = 12
        self.speed_x = 0
        self.speed_y = 0
        self.image = pygame.image.load("img/pilotagolf.png").convert_alpha()
        self.image = pygame.transform.smoothscale(self.image, (24, 24))

    def move(self, obstacles):
        global force, shooting

        self.x += self.speed_x
        self.y += self.speed_y

        friction = 0.98
        self.speed_x *= friction
        self.speed_y *= friction

        if abs(self.speed_x) < 0.1:
            self.speed_x = 0
        if abs(self.speed_y) < 0.1:
            self.speed_y = 0

        if self.x - self.radius < 0:
            self.x = self.radius
            self.speed_x = -self.speed_x * 0.5
            force = 0
        if self.x + self.radius > WIDTH:
            self.x = WIDTH - self.radius
            self.speed_x = -self.speed_x * 0.5
            force = 0
        if self.y - self.radius < 0:
            self.y = self.radius
            self.speed_y = -self.speed_y * 0.5
            force = 0
        if self.y + self.radius > HEIGHT:
            self.y = HEIGHT - self.radius
            self.speed_y = -self.speed_y * 0.5
            force = 0

        ball_rect = pygame.Rect(self.x - self.radius, self.y - self.radius, self.radius * 2, self.radius * 2)

        # Comprovar col·lisió amb obstacles
        for obstacle in obstacles:
            if obstacle.rect.colliderect(ball_rect):
                # Tornar la pilota al punt d'inici
                self.x, self.y = ball_start_pos
                self.speed_x = 0
                self.speed_y = 0
                force = 0
                shooting = False
                break  # Sortir del bucle al reposicionar

    def draw(self, screen):
        screen.blit(self.image, (self.x - self.radius, self.y - self.radius))

    def hit(self, power, angle):
        self.speed_x = power * math.cos(angle)
        self.speed_y = power * math.sin(angle)
        hit_sound.play()


class Obstacle:
    def __init__(self, x, y, width, height):
        self.rect = pygame.Rect(x, y, width, height)

    def draw(self, screen):
        pygame.draw.rect(screen, RED, self.rect)


class Confetti:
    def __init__(self):
        self.particles = []

    def add(self, x, y):
        for _ in range(30):
            self.particles.append([
                x, y,
                random.uniform(-5, 5), random.uniform(-5, 5),
                random.randint(2, 5)
            ])

    def update(self):
        for p in self.particles:
            p[0] += p[2]
            p[1] += p[3]
            p[4] -= 0.1
        self.particles = [p for p in self.particles if p[4] > 0]

    def draw(self, screen):
        for p in self.particles:
            color = (
                random.randint(100, 255),
                random.randint(100, 255),
                random.randint(100, 255)
            )
            pygame.draw.circle(screen, color, (int(p[0]), int(p[1])), int(p[4]))


# Nivells amb obstacles més complicats i distribuïts per tot el mapa, formant passadissos
levels = [
    {
        "goal": pygame.Rect(900, 500, 20, 20),
        "obstacles": [
            Obstacle(450, 250, 20, 250),
            Obstacle(500, 250, 100, 20),
            Obstacle(500, 480, 100, 20),
            Obstacle(600, 250, 20, 250),
            Obstacle(650, 300, 150, 20),
        ]
    },
    {
        "goal": pygame.Rect(1100, 150, 20, 20),
        "obstacles": [
            Obstacle(650, 350, 20, 140),
            Obstacle(650, 350, 150, 20),
            Obstacle(800, 350, 20, 130),
            Obstacle(650, 480, 150, 20),
            Obstacle(820, 400, 20, 130),
            Obstacle(900, 100, 100, 50),
            Obstacle(950, 200, 20, 100),
        ],
    },
    {
        "goal": pygame.Rect(1150, 620, 25, 25),
        "obstacles": [
            Obstacle(580, 400, 20, 180),
            Obstacle(610, 400, 150, 20),
            Obstacle(760, 420, 20, 160),
            Obstacle(790, 480, 150, 20),
            Obstacle(940, 430, 20, 230),
            Obstacle(960, 430, 150, 20),
            Obstacle(1110, 540, 20, 120),
            Obstacle(1140, 580, 100, 20),
        ],
    },
    {
        "goal": pygame.Rect(1200, 100, 30, 30),
        "obstacles": [
            Obstacle(480, 190, 20, 160),
            Obstacle(500, 190, 180, 20),
            Obstacle(690, 210, 20, 160),
            Obstacle(500, 360, 200, 20),
            Obstacle(710, 360, 20, 160),
            Obstacle(730, 360, 200, 20),
            Obstacle(930, 360, 20, 160),
            Obstacle(950, 390, 180, 20),
            Obstacle(1130, 390, 20, 180),
            Obstacle(1150, 560, 150, 20),
            Obstacle(1250, 460, 20, 80),
            Obstacle(780, 240, 110, 20),
            Obstacle(830, 240, 20, 110),
            Obstacle(980, 510, 20, 110),
            Obstacle(1000, 610, 110, 20),
            Obstacle(1050, 100, 20, 80),
        ],
    },
]

level = 0
ball = Ball(*ball_start_pos)
confetti = Confetti()
running = True
force = 0
max_force = 10

shots_count = 0
level_start_time = 0


def draw_force_bar(screen, force):
    bar_height = force * 40
    pygame.draw.rect(screen, BLUE, (20, 500 - bar_height, 20, bar_height))
    draw_text("FORÇA", 20, 510, WHITE)


def draw_level_indicator(screen, level):
    draw_text(f"Nivell: {level + 1}", WIDTH - 150, 20, WHITE)


def show_menu():
    screen.fill(BLACK)
    background_image = pygame.image.load("img/presentacio.png").convert()
    background_image = pygame.transform.smoothscale(background_image, (WIDTH, HEIGHT))
    screen.blit(background_image, (0, 0))
    draw_text("Golf Game", WIDTH // 2, HEIGHT // 3, WHITE, center=True)
    draw_text("Prem ENTER per començar", WIDTH // 2, HEIGHT // 2 + 60, WHITE, center=True)
    draw_text("Prem C per a Crèdits", WIDTH // 2, HEIGHT // 2 + 100, WHITE, center=True)


def show_credits():
    screen.fill(BLACK)
    draw_text("Crèdits", WIDTH // 2, HEIGHT // 3, WHITE, center=True)
    draw_text("Realitzat per: Pous i Moha", WIDTH // 2, HEIGHT // 2, WHITE, center=True)
    draw_text("Prem ESC per tornar al Menú", WIDTH // 2, HEIGHT // 2 + 40, WHITE, center=True)


def show_post_level():
    screen.fill(BLACK)
    total_time = level_end_time - level_start_time
    draw_text(f"Nivell completat!", WIDTH // 2, HEIGHT // 4, WHITE, center=True)
    draw_text(f"Temps: {total_time:.2f} segons", WIDTH // 2, HEIGHT // 4 + 40, WHITE, center=True)
    draw_text(f"Tirs: {shots_count}", WIDTH // 2, HEIGHT // 4 + 80, WHITE, center=True)
    draw_text("Prem R per repetir el nivell", WIDTH // 2, HEIGHT // 4 + 140, WHITE, center=True)
    if level < len(levels) - 1:
        draw_text("Prem N per següent nivell", WIDTH // 2, HEIGHT // 4 + 180, WHITE, center=True)
    else:
        draw_text("Prem N per acabar el joc", WIDTH // 2, HEIGHT // 4 + 180, WHITE, center=True)


while running:
    if game_state == "menu":
        show_menu()
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_RETURN:
                    game_state = "playing"
                    level_start_time = time.time()
                    shots_count = 0
                elif event.key == pygame.K_c:
                    game_state = "credits"

    elif game_state == "credits":
        show_credits()
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_ESCAPE:
                    game_state = "menu"

    elif game_state == "playing":
        screen.blit(grass_texture, (0, 0))
        pygame.draw.rect(screen, WHITE, levels[level]["goal"])
        for obstacle in levels[level]["obstacles"]:
            obstacle.draw(screen)
        ball.draw(screen)
        confetti.draw(screen)
        draw_force_bar(screen, force)
        draw_level_indicator(screen, level)

        if not shooting:
            mx, my = pygame.mouse.get_pos()
            pygame.draw.line(screen, WHITE, (ball.x, ball.y), (mx, my), 2)

        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.MOUSEBUTTONDOWN and not shooting:
                force = 0
            elif event.type == pygame.MOUSEBUTTONUP and not shooting and force > 0:
                mx, my = pygame.mouse.get_pos()
                dx, dy = mx - ball.x, my - ball.y
                angle = math.atan2(dy, dx)
                ball.hit(force, angle)
                shooting = True
                shots_count += 1
                force = 0
            elif event.type == pygame.MOUSEMOTION and pygame.mouse.get_pressed()[0] and not shooting:
                force = min(max_force, force + 0.2)

        ball.move(levels[level]["obstacles"])
        confetti.update()

        if shooting and ball.speed_x == 0 and ball.speed_y == 0 and not show_congrats:
            shooting = False

        if levels[level]["goal"].collidepoint(ball.x, ball.y) and not show_congrats:
            show_congrats = True
            congrats_time = time.time()
            confetti.add(ball.x, ball.y)
            shooting = False
            ball.speed_x = 0
            ball.speed_y = 0

        if show_congrats:
            draw_text("Felicitats! Has encistellat la pilota!", WIDTH // 2, HEIGHT // 2 - 30, WHITE, center=True)
            confetti.draw(screen)
            confetti.update()
            if time.time() - congrats_time > 3:
                show_congrats = False
                level_end_time = time.time()
                game_state = "postlevel"
                ball = Ball(*ball_start_pos)
                force = 0
                shooting = False

    elif game_state == "postlevel":
        show_post_level()
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_r:
                    ball = Ball(*ball_start_pos)
                    confetti = Confetti()
                    force = 0
                    shooting = False
                    show_congrats = False
                    game_state = "playing"
                    level_start_time = time.time()
                    shots_count = 0
                elif event.key == pygame.K_n:
                    level += 1
                    if level >= len(levels):
                        game_state = "endgame"
                    else:
                        ball = Ball(*ball_start_pos)
                        confetti = Confetti()
                        force = 0
                        shooting = False
                        show_congrats = False
                        game_state = "playing"
                        level_start_time = time.time()
                        shots_count = 0

    elif game_state == "endgame":
        screen.fill(BLACK)
        draw_text("Has acabat el joc!", WIDTH // 2, HEIGHT // 3, WHITE, center=True)
        draw_text("Prem M per tornar al menú principal", WIDTH // 2, HEIGHT // 2, WHITE, center=True)
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            elif event.type == pygame.KEYDOWN:
                if event.key == pygame.K_m:
                    game_state = "menu"
                    level = 0

    pygame.display.update()
    clock.tick(60)
