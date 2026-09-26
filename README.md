from OpenGL.GL import *
from OpenGL.GLUT import *
from OpenGL.GLUT.fonts import GLUT_BITMAP_HELVETICA_18
import random, time


W, H = 420, 640
ROAD_L, ROAD_R = 110, 310
LANE_X = [150, 210, 270]

player_x = 210.0
obstacles = []              # [x, y, color]
score = 0.0
game_over = False
last_time = time.time()
spawn_timer = 1.0

# Cohen-Sutherland Line Clipping
INSIDE, LEFT, RIGHT, BOTTOM, TOP = 0, 1, 2, 4, 8
CLIP_L, CLIP_R, CLIP_B, CLIP_T = ROAD_L, ROAD_R, 0, H

def out_code(x, y):
    # কী করছে: পয়েন্টটি কোন অঞ্চলে আছে তার region-code বের করছে
    # কেন লাগছে: Cohen-Sutherland ক্লিপিং-এ প্রতিটি এন্ডপয়েন্টের কোড দরকার
    # real world-এ এটা কোথায় দেখা যায়: GPU rendering pipeline-এ viewport clipping
    code = INSIDE
    if x < CLIP_L: code |= LEFT
    elif x > CLIP_R: code |= RIGHT
    if y < CLIP_B: code |= BOTTOM
    elif y > CLIP_T: code |= TOP
    return code

def cohen_sutherland_clip(x1, y1, x2, y2):
    # কী করছে: লাইনের যে অংশ ক্লিপ-উইন্ডোর বাইরে সেটা কেটে বাদ দিচ্ছে
    # কেন লাগছে: লেন-ডিভাইডার স্ক্রল করার সময় উপরে/নিচে বের হয়ে গেলে সঠিকভাবে কাটতে হবে
    # real world-এ: ম্যাপ/গেম রেন্ডারিং-এ অদৃশ্য geometry বাদ দেওয়া হয় এভাবেই
    c1, c2 = out_code(x1, y1), out_code(x2, y2)
    accept = False
    for _ in range(6):
        if c1 == 0 and c2 == 0:
            accept = True; break
        if c1 & c2:
            break
        cout = c1 if c1 else c2
        if cout & TOP:
            x = x1 + (x2 - x1) * (CLIP_T - y1) / (y2 - y1); y = CLIP_T
        elif cout & BOTTOM:
            x = x1 + (x2 - x1) * (CLIP_B - y1) / (y2 - y1); y = CLIP_B
        elif cout & RIGHT:
            y = y1 + (y2 - y1) * (CLIP_R - x1) / (x2 - x1); x = CLIP_R
        else:
            y = y1 + (y2 - y1) * (CLIP_L - x1) / (x2 - x1); x = CLIP_L
        if cout == c1:
            x1, y1 = x, y; c1 = out_code(x1, y1)
        else:
            x2, y2 = x, y; c2 = out_code(x2, y2)
    return accept, x1, y1, x2, y2

#Color-filled shapes 
def filled_poly(points, color):
    glColor3f(*color)
    glBegin(GL_POLYGON)
    for x, y in points:
        glVertex2f(x, y)
    glEnd()

def filled_rect(cx, cy, w, h, color):
    filled_poly([(cx-w/2,cy-h/2),(cx+w/2,cy-h/2),(cx+w/2,cy+h/2),(cx-w/2,cy+h/2)], color)

def draw_sidewalk():
    # কী করছে: রাস্তার বাইরের কালো অংশে ফুটপাতের টাইলের মতো প্যাটার্ন আঁকছে
    # কেন লাগছে: শুধু কালো ব্যাকগ্রাউন্ডের বদলে দৃশ্যত পূর্ণ ও বাস্তবসম্মত দেখাতে
    # real world-এ: শহরের রাস্তার পাশে ফুটপাতে এমন টাইল/পেভার ব্লক দেখা যায়
    tile = 20
    for x0 in (0, ROAD_R+40):
        x1 = x0 + (ROAD_L-40 if x0 == 0 else W-(ROAD_R+40))
        col_i = 0
        x = x0
        while x < x1:
            row_i = 0
            y = 0
            while y < H:
                shade = 0.58 if (col_i+row_i) % 2 == 0 else 0.50
                filled_rect(x+tile/2, y+tile/2, tile-2, tile-2, (shade,shade,shade+0.03))
                y += tile; row_i += 1
            x += tile; col_i += 1

def draw_road_and_barriers():
    draw_sidewalk()
    filled_rect(W/2, H/2, ROAD_R-ROAD_L, H, (0.35,0.35,0.35))
    for bx in (ROAD_L-20, ROAD_R+20):
        for i, y in enumerate(range(0, H+40, 40)):
            c = (0.05,0.05,0.05) if i % 2 == 0 else (1,1,1)   # black & white island
            filled_rect(bx, y, 40, 40, c)

def draw_lane_dashes(offset):
    glColor3f(1,1,1)
    glLineWidth(3)
    glBegin(GL_LINES)
    for lx in (180, 240):   # midpoints BETWEEN lane centers (150,210,270) — not on them
        y = -60
        while y < H+60:
            y1, y2 = y+offset, y+offset+26
            accepted, cx1, cy1, cx2, cy2 = cohen_sutherland_clip(lx, y1, lx, y2)
            if accepted:
                glVertex2f(cx1, cy1); glVertex2f(cx2, cy2)
            y += 70
    glEnd()

def draw_f1_car(cx, cy, body_color):
    # কী করছে: গাড়ির local shape (0,0)-কেন্দ্রিক এঁকে translate দিয়ে রাস্তার সঠিক জায়গায় বসাচ্ছে
    # কেন লাগছে: প্রতি ফ্রেমে গাড়ি নড়াচড়া করাতে হলে কোঅর্ডিনেট বদলাতে হয়—এটাই 2D transform
    # real world-এ: গেম ইঞ্জিনে প্রতিটি অবজেক্ট এভাবেই translate করে সরানো হয়
    glPushMatrix()
    glTranslatef(cx, cy, 0)
    nose, tail = 30, -30
    # front wing
    filled_rect(0, nose-2, 30, 5, (0.9,0.9,0.9))
    # narrow pointed nose cone
    filled_poly([(-4,nose-2),(4,nose-2),(0,nose+6)], body_color)
    # main body: narrow front -> wide cockpit -> tapered rear
    filled_poly([(-5,nose-2),(5,nose-2),(10,8),(10,-14),(6,tail),(-6,tail),(-10,-14),(-10,8)], body_color)
    # cockpit
    filled_rect(0, 4, 8, 14, (0.05,0.05,0.05))
    # side pods
    filled_rect(-12, -6, 6, 16, body_color)
    filled_rect(12, -6, 6, 16, body_color)
    # rear wing + struts
    filled_rect(0, tail-4, 30, 5, (0.9,0.9,0.9))
    filled_rect(-9, tail, 2, 8, (0.2,0.2,0.2))
    filled_rect(9, tail, 2, 8, (0.2,0.2,0.2))
    # exposed open wheels at four corners
    for wx, wy in ((-16,14),(16,14),(-16,-16),(16,-16)):
        filled_rect(wx, wy, 8, 16, (0.05,0.05,0.05))
    glPopMatrix()

def draw_text(x, y, s, color=(1,1,1)):
    glColor3f(*color)
    glRasterPos2f(x, y)
    for ch in s:
        glutBitmapCharacter(GLUT_BITMAP_HELVETICA_18, ord(ch))

#Game logic
def spawn_wave():
    # কী করছে: প্রতি wave-এ ৩টি লেনের মধ্যে সর্বোচ্চ ২টি লেনে গাড়ি বসাচ্ছে
    # কেন: সবসময় অন্তত একটা লেন খালি রাখা—নাহলে খেলোয়াড়ের বাঁচার সুযোগ থাকে না
    # real world-এ: গেম লেভেল-ডিজাইনে "fairness guarantee" এভাবেই রাখা হয়
    fill_count = random.choice([1, 1, 2])          # rarely block 2 lanes, never 3
    lanes = random.sample(LANE_X, fill_count)
    for lx in lanes:
        obstacles.append([lx, H+40, (0.85,0.1,0.1)])

def update(_val=0):
    global last_time, spawn_timer, score, game_over
    now = time.time(); dt = now - last_time; last_time = now
    if not game_over:
        score += dt*10
        spawn_timer -= dt
        if spawn_timer <= 0:
            spawn_wave()
            spawn_timer = max(0.9, 1.6 - score/500)   # keeps a safe vertical gap between waves
        for ob in obstacles:
            ob[1] -= 160*dt
        obstacles[:] = [o for o in obstacles if o[1] > -60]
        for ob in obstacles:
            if abs(ob[0]-player_x) < 30 and abs(ob[1]-70) < 40:
                game_over = True
    glutPostRedisplay()
    glutTimerFunc(16, update, 0)

scroll_offset = [0.0]
def display():
    glClear(GL_COLOR_BUFFER_BIT)
    scroll_offset[0] = (scroll_offset[0] + 4) % 70
    draw_road_and_barriers()
    draw_lane_dashes(scroll_offset[0])
    for ob in obstacles:
        draw_f1_car(ob[0], ob[1], ob[2])
    if not game_over:
        draw_f1_car(player_x, 70, (0.15,0.75,1.0))
    filled_rect(75, H-22, 140, 30, (0.05,0.05,0.05))
    draw_text(15, H-30, f"Score: {int(score)}", color=(0.2,1,0.3))
    if game_over:
        draw_text(110, H/2, "CRASHED! Press R")
    glutSwapBuffers()

def move_player(key, x=0, y=0):
    global player_x
    step = 60
    if key in (GLUT_KEY_LEFT, b'a', b'A'):
        player_x = max(LANE_X[0], player_x - step)
    elif key in (GLUT_KEY_RIGHT, b'd', b'D'):
        player_x = min(LANE_X[-1], player_x + step)

def keyboard_ascii(key, x, y):
    global game_over, obstacles, score, player_x
    if key in (b'a', b'A', b'd', b'D'):
        move_player(key)
    elif key in (b'r', b'R') and game_over:
        obstacles.clear(); score = 0.0; game_over = False
        player_x = 210.0

def init():
    glClearColor(0.05,0.05,0.08,1)
    glMatrixMode(GL_PROJECTION)
    glLoadIdentity()
    glOrtho(0, W, 0, H, -1, 1)
    glMatrixMode(GL_MODELVIEW)

def main():
    glutInit()
    glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGB)
    glutInitWindowSize(W, H)
    glutCreateWindow(b"Neon Lane Dash")
    init()
    glutDisplayFunc(display)
    glutKeyboardFunc(keyboard_ascii)
    glutSpecialFunc(move_player)
    glutTimerFunc(16, update, 0)
    glutMainLoop()

if __name__ == "__main__":
    main()
