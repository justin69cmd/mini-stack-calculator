# mini-stack-calculator
A console-based arithmetic expression evaluator built using the Stack data structure in C++. Supports operator precedence, bracket parsing, and decimal numbers — all without using any built-in expression evaluators.  


#include <iostream>
#include <stack>
#include <string>
#include <sstream>
#include <cmath>
#include <cctype>
#include <vector>
#include <stdexcept>

using namespace std;

// ─────────────────────────────────────────────
//  Utility: operator precedence
// ─────────────────────────────────────────────
int precedence(char op) {
    if (op == '+' || op == '-') return 1;
    if (op == '*' || op == '/') return 2;
    if (op == '^')              return 3;
    return 0;
}

bool isOperator(char c) {
    return c == '+' || c == '-' || c == '*' || c == '/' || c == '^';
}

bool isRightAssociative(char op) {
    return op == '^';
}

// ─────────────────────────────────────────────
//  Step 1: Tokenise the infix expression
//  Handles multi-digit numbers and unary minus
// ─────────────────────────────────────────────
struct Token {
    enum Type { NUMBER, OPERATOR, LPAREN, RPAREN } type;
    double   number;
    char     op;
};

vector<Token> tokenise(const string& expr) {
    vector<Token> tokens;
    int i = 0, n = expr.size();

    while (i < n) {
        if (isspace(expr[i])) { ++i; continue; }

        // Number (integer or decimal)
        if (isdigit(expr[i]) || (expr[i] == '.' && i + 1 < n && isdigit(expr[i+1]))) {
            size_t pos;
            double val = stod(expr.substr(i), &pos);
            tokens.push_back({Token::NUMBER, val, 0});
            i += pos;
            continue;
        }

        // Unary minus: treat as "0 - ..."
        if (expr[i] == '-' &&
            (tokens.empty() ||
             tokens.back().type == Token::LPAREN ||
             tokens.back().type == Token::OPERATOR)) {
            tokens.push_back({Token::NUMBER, 0.0, 0});
        }

        if (expr[i] == '(') tokens.push_back({Token::LPAREN,  0, '('});
        else if (expr[i] == ')') tokens.push_back({Token::RPAREN,  0, ')'});
        else if (isOperator(expr[i])) tokens.push_back({Token::OPERATOR, 0, expr[i]});
        else throw runtime_error(string("Unknown character: ") + expr[i]);
        ++i;
    }
    return tokens;
}

// ─────────────────────────────────────────────
//  Step 2: Shunting-Yard → Postfix (RPN) tokens
// ─────────────────────────────────────────────
vector<Token> toPostfix(const vector<Token>& tokens) {
    vector<Token> output;
    stack<Token>  ops;

    for (const Token& t : tokens) {
        if (t.type == Token::NUMBER) {
            output.push_back(t);
        }
        else if (t.type == Token::OPERATOR) {
            while (!ops.empty() &&
                   ops.top().type == Token::OPERATOR &&
                   ((precedence(ops.top().op) > precedence(t.op)) ||
                    (precedence(ops.top().op) == precedence(t.op) && !isRightAssociative(t.op)))) {
                output.push_back(ops.top());
                ops.pop();
            }
            ops.push(t);
        }
        else if (t.type == Token::LPAREN) {
            ops.push(t);
        }
        else if (t.type == Token::RPAREN) {
            while (!ops.empty() && ops.top().type != Token::LPAREN) {
                output.push_back(ops.top());
                ops.pop();
            }
            if (ops.empty()) throw runtime_error("Mismatched parentheses!");
            ops.pop(); // discard '('
        }
    }

    while (!ops.empty()) {
        if (ops.top().type == Token::LPAREN)
            throw runtime_error("Mismatched parentheses!");
        output.push_back(ops.top());
        ops.pop();
    }
    return output;
}

// ─────────────────────────────────────────────
//  Step 3: Evaluate Postfix using a value stack
// ─────────────────────────────────────────────
double evalPostfix(const vector<Token>& postfix) {
    stack<double> vals;

    for (const Token& t : postfix) {
        if (t.type == Token::NUMBER) {
            vals.push(t.number);
        } else {
            if (vals.size() < 2)
                throw runtime_error("Invalid expression!");
            double b = vals.top(); vals.pop();
            double a = vals.top(); vals.pop();

            switch (t.op) {
                case '+': vals.push(a + b); break;
                case '-': vals.push(a - b); break;
                case '*': vals.push(a * b); break;
                case '/':
                    if (b == 0) throw runtime_error("Division by zero!");
                    vals.push(a / b);
                    break;
                case '^': vals.push(pow(a, b)); break;
                default:  throw runtime_error("Unknown operator!");
            }
        }
    }

    if (vals.size() != 1) throw runtime_error("Invalid expression!");
    return vals.top();
}

// ─────────────────────────────────────────────
//  Debug helper: print postfix tokens
// ─────────────────────────────────────────────
void printPostfix(const vector<Token>& postfix) {
    cout << "  Postfix (RPN) : ";
    for (const Token& t : postfix) {
        if (t.type == Token::NUMBER) cout << t.number << " ";
        else                         cout << t.op    << " ";
    }
    cout << "\n";
}

// ─────────────────────────────────────────────
//  Main: evaluate a single expression string
// ─────────────────────────────────────────────
void evaluate(const string& expr) {
    cout << "\n  Expression     : " << expr << "\n";
    try {
        vector<Token> tokens  = tokenise(expr);
        vector<Token> postfix = toPostfix(tokens);
        printPostfix(postfix);
        double result = evalPostfix(postfix);

        // Print neatly: integer if no fractional part
        if (result == (long long)result)
            cout << "  Result         : " << (long long)result << "\n";
        else
            cout << "  Result         : " << result << "\n";
    } catch (const exception& e) {
        cout << "  Error          : " << e.what() << "\n";
    }
}

// ─────────────────────────────────────────────
//  REPL: interactive calculator loop
// ─────────────────────────────────────────────
void runInteractive() {
    cout << "\n";
    cout << "╔══════════════════════════════════════════╗\n";
    cout << "║   Mini Stack Calculator  (C++)           ║\n";
    cout << "║   Supports: + - * / ^ and ( )            ║\n";
    cout << "║   Type 'demo' for examples  |  'q' quit  ║\n";
    cout << "╚══════════════════════════════════════════╝\n";

    string line;
    while (true) {
        cout << "\n>> ";
        if (!getline(cin, line)) break;

        // Trim
        size_t s = line.find_first_not_of(" \t");
        if (s == string::npos) continue;
        line = line.substr(s);

        if (line == "q" || line == "quit" || line == "exit") {
            cout << "  Goodbye!\n\n";
            break;
        }

        if (line == "demo") {
            cout << "\n  ── Demo expressions ──";
            evaluate("3 + 5 * 2");
            evaluate("(3 + 5) * 2");
            evaluate("2 ^ 10");
            evaluate("100 / (4 + (3 * 2))");
            evaluate("-5 + 20");
            evaluate("3.5 * 2 + 1.5");
            evaluate("10 / 0");
            evaluate("(2 + 3");
            continue;
        }

        evaluate(line);
    }
}

int main() {
    runInteractive();
    return 0;
}
