```
#include <windows.h>
#include <stdbool.h>
#include <string.h>

#define WINDOW_WIDTH  800
#define WINDOW_HEIGHT 600

// 무지개 색상 정의 (빨, 주, 노, 초, 파, 남, 보)
COLORREF rainbowColors[7] = {
    RGB(255, 0, 0),     // 빨강
    RGB(255, 127, 0),   // 주황
    RGB(255, 255, 0),   // 노랑
    RGB(0, 255, 0),     // 초록
    RGB(0, 0, 255),     // 파랑
    RGB(75, 0, 130),    // 남색
    RGB(148, 0, 211)    // 보라
};

// 게임 객체 구조체
typedef struct {
    float x, y;
    float dx, dy;
    int radius;
} Ball;

typedef struct {
    float x, y;
    float width, height;
    float speed;
} Paddle;

typedef struct {
    float x, y;
    float width, height;
    bool active;
    int colorIndex;
} Brick;

#define BRICK_ROWS 7
#define BRICK_COLS 10
#define BRICK_WIDTH 70
#define BRICK_HEIGHT 20
#define BRICK_PADDING 8
#define BRICK_OFFSET_X 40
#define BRICK_OFFSET_Y 50

Ball ball;
Paddle paddle;
Brick bricks[BRICK_ROWS][BRICK_COLS];

bool isLeftPressed = false;
bool isRightPressed = false;
bool gameOver = false;
bool gameWin = false;
int score = 0;

void InitGame() {
    // 공 초기화
    ball.x = WINDOW_WIDTH / 2.0f;
    ball.y = WINDOW_HEIGHT / 2.0f;
    ball.dx = 4.0f;
    ball.dy = -4.0f;
    ball.radius = 8;

    // 패들 초기화
    paddle.width = 110;
    paddle.height = 15;
    paddle.x = (WINDOW_WIDTH - paddle.width) / 2.0f;
    paddle.y = WINDOW_HEIGHT - 60;
    paddle.speed = 9.0f;

    // 무지개 벽돌 초기화 (행마다 무지개 색상 매칭)
    for (int i = 0; i < BRICK_ROWS; i++) {
        for (int j = 0; j < BRICK_COLS; j++) {
            bricks[i][j].x = BRICK_OFFSET_X + j * (BRICK_WIDTH + BRICK_PADDING);
            bricks[i][j].y = BRICK_OFFSET_Y + i * (BRICK_HEIGHT + BRICK_PADDING);
            bricks[i][j].width = BRICK_WIDTH;
            bricks[i][j].height = BRICK_HEIGHT;
            bricks[i][j].active = true;
            bricks[i][j].colorIndex = i; 
        }
    }
    score = 0;
    gameOver = false;
    gameWin = false;
}

void UpdateGame() {
    if (gameOver || gameWin) return;

    // 패들 이동 처리
    if (isLeftPressed) {
        paddle.x -= paddle.speed;
        if (paddle.x < 0) paddle.x = 0;
    }
    if (isRightPressed) {
        paddle.x += paddle.speed;
        if (paddle.x > WINDOW_WIDTH - paddle.width) paddle.x = WINDOW_WIDTH - paddle.width;
    }

    // 공 이동
    ball.x += ball.dx;
    ball.y += ball.dy;

    // 벽 충돌 (좌우)
    if (ball.x - ball.radius < 0) {
        ball.x = ball.radius;
        ball.dx = -ball.dx;
    }
    if (ball.x + ball.radius > WINDOW_WIDTH) {
        ball.x = WINDOW_WIDTH - ball.radius;
        ball.dx = -ball.dx;
    }
    // 벽 충돌 (상단)
    if (ball.y - ball.radius < 0) {
        ball.y = ball.radius;
        ball.dy = -ball.dy;
    }

    // 바닥 충돌 (게임 오버)
    if (ball.y + ball.radius > WINDOW_HEIGHT) {
        gameOver = true;
    }

    // 패들 충돌 처리
    if (ball.y + ball.radius >= paddle.y &&
        ball.y - ball.radius <= paddle.y + paddle.height &&
        ball.x >= paddle.x &&
        ball.x <= paddle.x + paddle.width) {
        
        ball.y = paddle.y - ball.radius;
        ball.dy = -ball.dy;
        
        // 패들의 맞은 위치에 따라 반사 각도 조절 (재미 요소)
        float hitPoint = (ball.x - (paddle.x + paddle.width / 2.0f)) / (paddle.width / 2.0f);
        ball.dx = hitPoint * 5.5f;
    }

    // 벽돌 충돌 검사
    bool allCleared = true;
    for (int i = 0; i < BRICK_ROWS; i++) {
        for (int j = 0; j < BRICK_COLS; j++) {
            Brick* b = &bricks[i][j];
            if (b->active) {
                allCleared = false;
                // 사각형 대 사각형(AABB) 충돌 판정
                if (ball.x + ball.radius >= b->x &&
                    ball.x - ball.radius <= b->x + b->width &&
                    ball.y + ball.radius >= b->y &&
                    ball.y - ball.radius <= b->y + b->height) {
                    
                    b->active = false;
                    ball.dy = -ball.dy;
                    score += 10;
                }
            }
        }
    }

    if (allCleared) {
        gameWin = true;
    }
}

LRESULT CALLBACK WndProc(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam) {
    switch (message) {
    case WM_CREATE:
        InitGame();
        SetTimer(hwnd, 1, 16, NULL); // 약 60 FPS 유지
        break;

    case WM_KEYDOWN:
        if (wParam == VK_LEFT) isLeftPressed = true;
        if (wParam == VK_RIGHT) isRightPressed = true;
        // R 키를 누르면 게임 재시작
        if (wParam == 'R' || wParam == 'r') {
            if (gameOver || gameWin) {
                InitGame();
                InvalidateRect(hwnd, NULL, FALSE);
            }
        }
        break;

    case WM_KEYUP:
        if (wParam == VK_LEFT) isLeftPressed = false;
        if (wParam == VK_RIGHT) isRightPressed = false;
        break;

    case WM_TIMER:
        UpdateGame();
        InvalidateRect(hwnd, NULL, FALSE);
        break;

    case WM_PAINT: {
        PAINTSTRUCT ps;
        HDC hdc = BeginPaint(hwnd, &ps);

        // 더블 버퍼링 설정 (화면 깜빡임 방지)
        HDC memDC = CreateCompatibleDC(hdc);
        HBITMAP memBitmap = CreateCompatibleBitmap(hdc, WINDOW_WIDTH, WINDOW_HEIGHT);
        HBITMAP oldBitmap = (HBITMAP)SelectObject(memDC, memBitmap);

        // 배경 채우기 (어두운 감성 배경)
        HBRUSH bgBrush = CreateSolidBrush(RGB(15, 15, 25));
        RECT rect = { 0, 0, WINDOW_WIDTH, WINDOW_HEIGHT };
        FillRect(memDC, &rect, bgBrush);
        DeleteObject(bgBrush);

        // 무지개 벽돌 그리기
        for (int i = 0; i < BRICK_ROWS; i++) {
            for (int j = 0; j < BRICK_COLS; j++) {
                if (bricks[i][j].active) {
                    HBRUSH brickBrush = CreateSolidBrush(rainbowColors[bricks[i][j].colorIndex]);
                    HBRUSH oldBrush = (HBRUSH)SelectObject(memDC, brickBrush);
                    Rectangle(memDC, (int)bricks[i][j].x, (int)bricks[i][j].y,
                              (int)(bricks[i][j].x + bricks[i][j].width),
                              (int)(bricks[i][j].y + bricks[i][j].height));
                    SelectObject(memDC, oldBrush);
                    DeleteObject(brickBrush);
                }
            }
        }

        // 패들 그리기 (흰색)
        HBRUSH paddleBrush = CreateSolidBrush(RGB(255, 255, 255));
        HBRUSH oldPaddleBrush = (HBRUSH)SelectObject(memDC, paddleBrush);
        Rectangle(memDC, (int)paddle.x, (int)paddle.y,
                  (int)(paddle.x + paddle.width), (int)(paddle.y + paddle.height));
        SelectObject(memDC, oldPaddleBrush);
        DeleteObject(paddleBrush);

        // 공 그리기 (노란색)
        HBRUSH ballBrush = CreateSolidBrush(RGB(255, 255, 0));
        HBRUSH oldBallBrush = (HBRUSH)SelectObject(memDC, ballBrush);
        Ellipse(memDC, (int)(ball.x - ball.radius), (int)(ball.y - ball.radius),
                (int)(ball.x + ball.radius), (int)(ball.y + ball.radius));
        SelectObject(memDC, oldBallBrush);
        DeleteObject(ballBrush);

        // 상단 점수 표시
        SetBkMode(memDC, TRANSPARENT);
        SetTextColor(memDC, RGB(255, 255, 255));
        char scoreText[64];
        wsprintfA(scoreText, "Score: %d", score);
        TextOutA(memDC, 20, 20, scoreText, (int)strlen(scoreText));

        // 게임 오버 또는 승리 메시지 출력
        if (gameOver) {
            SetTextColor(memDC, RGB(255, 60, 60));
            const char* msg = "GAME OVER! - Press 'R' to Restart";
            TextOutA(memDC, WINDOW_WIDTH / 2 - 110, WINDOW_HEIGHT / 2, msg, (int)strlen(msg));
        } else if (gameWin) {
            SetTextColor(memDC, RGB(60, 255, 60));
            const char* msg = "VICTORY! - Press 'R' to Restart";
            TextOutA(memDC, WINDOW_WIDTH / 2 - 100, WINDOW_HEIGHT / 2, msg, (int)strlen(msg));
        }

        // 메모리 DC에 그려진 화면을 실제 화면 DC로 한 번에 복사
        BitBlt(hdc, 0, 0, WINDOW_WIDTH, WINDOW_HEIGHT, memDC, 0, 0, SRCCOPY);

        // 리소스 정리 (메모리 누수 방지)
        SelectObject(memDC, oldBitmap);
        DeleteObject(memBitmap);
        DeleteDC(memDC);

        EndPaint(hwnd, &ps);
        break;
    }

    case WM_DESTROY:
        KillTimer(hwnd, 1);
        PostQuitMessage(0);
        break;

    default:
        return DefWindowProc(hwnd, message, wParam, lParam);
    }
    return 0;
}

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow) {
    WNDCLASSA wc = { 0 };
    wc.lpfnWndProc = WndProc;
    wc.hInstance = hInstance;
    wc.lpszClassName = "RainbowBreakoutClass";
    wc.hCursor = LoadCursor(NULL, IDC_ARROW);
    wc.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);

    if (!RegisterClassA(&wc)) {
        MessageBoxA(NULL, "Window Registration Failed!", "Error", MB_ICONEXCLAMATION | MB_OK);
        return 0;
    }

    // 창 크기 고정 및 실행
    HWND hwnd = CreateWindowA(
        "RainbowBreakoutClass",
        "무지개 블록깨기 게임",
        WS_OVERLAPPEDWINDOW & ~WS_MAXIMIZEBOX & ~WS_THICKFRAME,
        CW_USEDEFAULT, CW_USEDEFAULT,
        WINDOW_WIDTH + 16, WINDOW_HEIGHT + 39,
        NULL, NULL, hInstance, NULL
    );

    if (hwnd == NULL) {
        MessageBoxA(NULL, "Window Creation Failed!", "Error", MB_ICONEXCLAMATION | MB_OK);
        return 0;
    }

    ShowWindow(hwnd, nCmdShow);
    UpdateWindow(hwnd);

    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0) > 0) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }

    return (int)msg.wParam;
}
```
