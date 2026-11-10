#include <windows.h>
#include <string>
using namespace std;

/* ================= GLOBALS ================= */
HINSTANCE hInst;
HWND hQuestion, hOpt1, hOpt2, hOpt3, hTimer;
HWND hUser, hPass, hLoginBtn, hUserLbl, hPassLbl;
HFONT hFont;

int timeLeft = 15;
int totalQuestionsAllowed = 3;
int questionsAnswered = 0;

/* ================= APP STATE ================= */
enum AppState { LOGIN, QUIZ };
AppState currentState = LOGIN;

/* ================= IDS ================= */
#define ID_O1 101
#define ID_O2 102
#define ID_O3 103
#define ID_NEXT 104
#define ID_LOGIN 201

/* ================= DSA ================= */
// Linked List Node
struct QuestionNode {
    wstring q, opt[3];
    int correct;
    QuestionNode* next;
};

// Queue
class QuestionQueue {
public:
    QuestionNode* f = NULL;
    QuestionNode* r = NULL;
    void enqueue(QuestionNode* n) {
        if (!r) f = r = n;
        else { r->next = n; r = n; }
    }
    QuestionNode* dequeue() {
        if (!f) return NULL;
        QuestionNode* t = f;
        f = f->next;
        return t;
    }
} quizQueue;

// Stack
class ScoreStack {
    int a[50], top = -1;
public:
    void push(int x) { a[++top] = x; }
    int total() {
        int s = 0;
        for (int i = 0; i <= top; i++) s += a[i];
        return s;
    }
} scoreStack;

QuestionNode* currentQ;

/* ================= HASH LOGIN ================= */
#define SIZE 10
struct User { wstring u, p; };
User hashT[SIZE];

int hashFunc(wstring k) { return k.length() % SIZE; }

void initUsers() {
    hashT[hashFunc(L"admin")] = { L"admin", L"123" };
}

bool checkLogin(wstring u, wstring p) {
    int i = hashFunc(u);
    return hashT[i].u == u && hashT[i].p == p;
}

/* ================= QUESTIONS ================= */
QuestionNode* makeQ(wstring q, wstring a, wstring b, wstring c, int cor) {
    QuestionNode* n = new QuestionNode;
    n->q = q;
    n->opt[0] = a; n->opt[1] = b; n->opt[2] = c;
    n->correct = cor;
    n->next = NULL;
    return n;
}

void loadQuestions() {
    quizQueue.enqueue(makeQ(L"Capital of Pakistan?", L"Karachi", L"Islamabad", L"Lahore", 1));
    quizQueue.enqueue(makeQ(L"FIFO DS?", L"Stack", L"Queue", L"Tree", 1));
    quizQueue.enqueue(makeQ(L"LIFO DS?", L"Queue", L"Stack", L"Array", 1));
    quizQueue.enqueue(makeQ(L"Hash search time?", L"O(n)", L"O(log n)", L"O(1)", 2));
    quizQueue.enqueue(makeQ(L"Non‑Linear DS?", L"Array", L"Linked List", L"Tree", 2));
}

/* ================= UI HELPERS ================= */
void hideLoginUI() {
    ShowWindow(hUser, SW_HIDE);
    ShowWindow(hPass, SW_HIDE);
    ShowWindow(hLoginBtn, SW_HIDE);
    ShowWindow(hUserLbl, SW_HIDE);
    ShowWindow(hPassLbl, SW_HIDE);
}

void showQuizUI(HWND hWnd) {
    hQuestion = CreateWindow(L"STATIC", L"", WS_VISIBLE | WS_CHILD,
        30, 20, 440, 40, hWnd, 0, hInst, 0);

    hOpt1 = CreateWindow(L"BUTTON", L"", WS_VISIBLE | WS_CHILD | BS_AUTORADIOBUTTON,
        30, 80, 300, 25, hWnd, (HMENU)ID_O1, hInst, 0);

    hOpt2 = CreateWindow(L"BUTTON", L"", WS_VISIBLE | WS_CHILD | BS_AUTORADIOBUTTON,
        30, 110, 300, 25, hWnd, (HMENU)ID_O2, hInst, 0);

    hOpt3 = CreateWindow(L"BUTTON", L"", WS_VISIBLE | WS_CHILD | BS_AUTORADIOBUTTON,
        30, 140, 300, 25, hWnd, (HMENU)ID_O3, hInst, 0);

    CreateWindow(L"BUTTON", L"Next", WS_VISIBLE | WS_CHILD,
        30, 180, 100, 30, hWnd, (HMENU)ID_NEXT, hInst, 0);

    hTimer = CreateWindow(L"STATIC", L"Time: 15",
        WS_VISIBLE | WS_CHILD, 360, 20, 120, 25, hWnd, 0, hInst, 0);
}

void startTimer(HWND hWnd) {
    timeLeft = 15;
    SetTimer(hWnd, 1, 1000, NULL);
}

void showQuestion(HWND hWnd) {
    if (questionsAnswered == totalQuestionsAllowed) {
        wchar_t r[100];
        wsprintf(r, L"Quiz Finished!\nScore: %d", scoreStack.total());
        MessageBox(hWnd, r, L"Result", MB_OK);
        DestroyWindow(hWnd);
        return;
    }

    currentQ = quizQueue.dequeue();
    questionsAnswered++;

    SetWindowText(hQuestion, currentQ->q.c_str());
    SetWindowText(hOpt1, currentQ->opt[0].c_str());
    SetWindowText(hOpt2, currentQ->opt[1].c_str());
    SetWindowText(hOpt3, currentQ->opt[2].c_str());

    SendMessage(hOpt1, BM_SETCHECK, BST_UNCHECKED, 0);
    SendMessage(hOpt2, BM_SETCHECK, BST_UNCHECKED, 0);
    SendMessage(hOpt3, BM_SETCHECK, BST_UNCHECKED, 0);

    startTimer(hWnd);
}

/* ================= WINDOW ================= */
LRESULT CALLBACK WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam) {

    switch (msg) {

    case WM_CREATE:
        initUsers();
        loadQuestions();

        hFont = CreateFont(20, 0, 0, 0, FW_BOLD, 0, 0, 0,
            DEFAULT_CHARSET, 0, 0, 0, 0, L"Segoe UI");

        hUserLbl = CreateWindow(L"STATIC", L"Username:",
            WS_VISIBLE | WS_CHILD, 160, 80, 80, 25, hWnd, 0, hInst, 0);

        hUser = CreateWindow(L"EDIT", L"",
            WS_VISIBLE | WS_CHILD | WS_BORDER,
            250, 80, 120, 25, hWnd, 0, hInst, 0);

        hPassLbl = CreateWindow(L"STATIC", L"Password:",
            WS_VISIBLE | WS_CHILD, 160, 120, 80, 25, hWnd, 0, hInst, 0);

        hPass = CreateWindow(L"EDIT", L"",
            WS_VISIBLE | WS_CHILD | WS_BORDER | ES_PASSWORD,
            250, 120, 120, 25, hWnd, 0, hInst, 0);

        hLoginBtn = CreateWindow(L"BUTTON", L"Login",
            WS_VISIBLE | WS_CHILD,
            250, 160, 80, 30, hWnd, (HMENU)ID_LOGIN, hInst, 0);
        break;

    case WM_COMMAND:
        if (LOWORD(wParam) == ID_LOGIN) {
            wchar_t u[50], p[50];
            GetWindowText(hUser, u, 50);
            GetWindowText(hPass, p, 50);

            if (checkLogin(u, p)) {
                hideLoginUI();
                currentState = QUIZ;
                showQuizUI(hWnd);
                showQuestion(hWnd);
            }
            else MessageBox(hWnd, L"Invalid Login", L"Error", MB_ICONERROR);
        }

        if (LOWORD(wParam) == ID_NEXT) {
            KillTimer(hWnd, 1);

            if (SendMessage(hOpt1, BM_GETCHECK, 0, 0) && currentQ->correct == 0) scoreStack.push(1);
            else if (SendMessage(hOpt2, BM_GETCHECK, 0, 0) && currentQ->correct == 1) scoreStack.push(1);
            else if (SendMessage(hOpt3, BM_GETCHECK, 0, 0) && currentQ->correct == 2) scoreStack.push(1);
            else scoreStack.push(0);

            showQuestion(hWnd);
        }
        break;

    case WM_TIMER:
        timeLeft--;
        wchar_t t[20];
        wsprintf(t, L"Time: %d", timeLeft);
        SetWindowText(hTimer, t);
        if (timeLeft == 0) SendMessage(hWnd, WM_COMMAND, ID_NEXT, 0);
        break;

    case WM_DESTROY:
        PostQuitMessage(0);
        break;
    }
    return DefWindowProc(hWnd, msg, wParam, lParam);
}

/* ================= MAIN ================= */
int APIENTRY wWinMain(HINSTANCE hInstance, HINSTANCE, LPWSTR, int nCmdShow) {
    hInst = hInstance;

    WNDCLASS wc = {};
    wc.lpfnWndProc = WndProc;
    wc.hInstance = hInstance;
    wc.hbrBackground = (HBRUSH)(COLOR_WINDOW + 1);
    wc.lpszClassName = L"DSAQuizApp";
    RegisterClass(&wc);

    HWND hWnd = CreateWindow(wc.lpszClassName, L"DSA Quiz Application",
        WS_OVERLAPPEDWINDOW, CW_USEDEFAULT, 0, 520, 300,
        NULL, NULL, hInstance, NULL);

    ShowWindow(hWnd, nCmdShow);

    MSG msg;
    while (GetMessage(&msg, NULL, 0, 0)) {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    return 0;
}
