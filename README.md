# Bus-route-fare
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <math.h>

#define FILE_NAME "routes.txt"
#define LEN 100

typedef struct StopNode {
    char name[LEN];
    double fee;
    struct StopNode *next;
} StopNode;

typedef struct RouteNode {
    char routeName[LEN];
    StopNode *stopHead;
    struct RouteNode *next;
} RouteNode;

typedef struct FeeLog {            /* stack node for Undo */
    StopNode *stop;
    double oldFee;
    struct FeeLog *below;
} FeeLog;

RouteNode *head = NULL;
FeeLog *top = NULL;

/* ---------- helpers ---------- */
int equalsIgnoreCase(const char *a, const char *b) {
    while (*a && *b) {
        if (tolower((unsigned char)*a) != tolower((unsigned char)*b)) return 0;
        a++; b++;
    }
    return *a == *b;
}

int startsWithIgnoreCase(const char *s, const char *prefix) {
    while (*prefix) {
        if (!*s || tolower((unsigned char)*s) != tolower((unsigned char)*prefix)) return 0;
        s++; prefix++;
    }
    return 1;
}

void ask(const char *msg, char *buf) {
    printf("%s", msg);
    if (!fgets(buf, LEN, stdin)) { buf[0] = '\0'; return; }
    buf[strcspn(buf, "\r\n")] = '\0';
    /* trim leading spaces */
    char *p = buf;
    while (*p == ' ') p++;
    if (p != buf) memmove(buf, p, strlen(p) + 1);
    /* trim trailing spaces */
    size_t n = strlen(buf);
    while (n > 0 && buf[n - 1] == ' ') buf[--n] = '\0';
}

double askFee(const char *msg) {
    char buf[LEN], *end;
    ask(msg, buf);
    double f = strtod(buf, &end);
    if (buf[0] != '\0' && *end == '\0' && f > 0) return f;
    printf("Invalid fee. Fee must be a number greater than 0.\n");
    return -1;
}

RouteNode *findRoute(const char *name) {
    for (RouteNode *r = head; r; r = r->next)
        if (equalsIgnoreCase(r->routeName, name)) return r;
    return NULL;
}

RouteNode *findRouteByNumber(const char *text) {   /* "1" -> "Route 1 - ..." */
    char prefix[LEN];
    snprintf(prefix, LEN, "Route %s ", text);
    for (RouteNode *r = head; r; r = r->next)
        if (equalsIgnoreCase(r->routeName, text) || startsWithIgnoreCase(r->routeName, prefix))
            return r;
    return NULL;
}

StopNode *search(const char *key) {
    for (RouteNode *r = head; r; r = r->next)
        for (StopNode *t = r->stopHead; t; t = t->next)
            if (equalsIgnoreCase(t->name, key)) return t;
    return NULL;
}

RouteNode *routeOf(StopNode *s) {
    for (RouteNode *r = head; r; r = r->next)
        for (StopNode *t = r->stopHead; t; t = t->next)
            if (t == s) return r;
    return NULL;
}

/* ---------- operations ---------- */
RouteNode *addRoute(const char *name) {
    RouteNode *n = (RouteNode *)malloc(sizeof(RouteNode));
    strncpy(n->routeName, name, LEN - 1);
    n->routeName[LEN - 1] = '\0';
    n->stopHead = NULL;
    n->next = NULL;
    if (head == NULL) { head = n; return n; }
    RouteNode *t = head;
    while (t->next) t = t->next;
    t->next = n;
    return n;
}

void addStop(RouteNode *r, const char *name, double fee) {
    StopNode *n = (StopNode *)malloc(sizeof(StopNode));
    strncpy(n->name, name, LEN - 1);
    n->name[LEN - 1] = '\0';
    n->fee = fee;
    n->next = NULL;
    if (r->stopHead == NULL) { r->stopHead = n; return; }
    StopNode *t = r->stopHead;
    while (t->next) t = t->next;
    t->next = n;
}

void addRouteOrStop(void) {
    char rn[LEN], sn[LEN];
    ask("Route name (e.g. Route 4 - Thuraiyur): ", rn);
    if (rn[0] == '\0') { printf("Route name cannot be empty.\n"); return; }
    RouteNode *r = findRoute(rn);
    if (r == NULL) { r = addRoute(rn); printf("New route added.\n"); }
    ask("Stop name (blank to skip): ", sn);
    if (sn[0] == '\0') return;
    if (search(sn)) { printf("Stop already exists.\n"); return; }
    double fee = askFee("Annual fee (Rs.): ");
    if (fee < 0) return;
    addStop(r, sn, fee);
    printf("Stop added.\n");
}

void display(void) {
    if (head == NULL) { printf("No routes available.\n"); return; }
    for (RouteNode *r = head; r; r = r->next) {
        printf("\n%s\n", r->routeName);
        if (r->stopHead == NULL) printf("   (no stops)\n");
        for (StopNode *t = r->stopHead; t; t = t->next)
            printf("   %-20s Rs. %.2f\n", t->name, t->fee);
    }
}

void searchStop(void) {
    char key[LEN];
    ask("Enter stop name: ", key);
    StopNode *t = search(key);
    if (!t) { printf("No match found.\n"); return; }
    printf("Route : %s\n", routeOf(t)->routeName);
    printf("Fee   : Rs. %.2f\n", t->fee);
}

void updateFee(void) {
    char key[LEN];
    ask("Enter stop name: ", key);
    StopNode *t = search(key);
    if (!t) { printf("No match found.\n"); return; }
    double fee = askFee("New fee (Rs.): ");
    if (fee < 0) return;
    FeeLog *log = (FeeLog *)malloc(sizeof(FeeLog));   /* push old fee */
    log->stop = t; log->oldFee = t->fee; log->below = top;
    top = log;
    t->fee = fee;
    printf("Fee updated.\n");
}

void deleteStop(void) {
    char key[LEN];
    ask("Enter stop name to delete: ", key);
    for (RouteNode *r = head; r; r = r->next) {
        if (r->stopHead == NULL) continue;
        if (equalsIgnoreCase(r->stopHead->name, key)) {
            StopNode *del = r->stopHead;
            r->stopHead = del->next;
            free(del);
            printf("Stop deleted.\n"); return;
        }
        StopNode *prev = r->stopHead, *cur = prev->next;
        while (cur && !equalsIgnoreCase(cur->name, key)) { prev = cur; cur = cur->next; }
        if (cur) {
            prev->next = cur->next;                  /* skip the deleted node */
            free(cur);
            printf("Stop deleted.\n"); return;
        }
    }
    printf("No match found.\n");
}

void countStops(void) {
    int c = 0;
    for (RouteNode *r = head; r; r = r->next)
        for (StopNode *t = r->stopHead; t; t = t->next) c++;
    printf("Total stops: %d\n", c);
}

void highestFee(void) {
    StopNode *best = NULL;
    for (RouteNode *r = head; r; r = r->next)
        for (StopNode *t = r->stopHead; t; t = t->next)
            if (!best || t->fee > best->fee) best = t;
    if (!best) { printf("No stops available.\n"); return; }
    printf("Highest fee: %s (%s) - Rs. %.2f\n", best->name, routeOf(best)->routeName, best->fee);
}

void reverseRoute(void) {
    char key[LEN];
    ask("Route number or name: ", key);
    RouteNode *r = findRouteByNumber(key);
    if (!r) { printf("Route not found.\n"); return; }
    StopNode *prev = NULL, *cur = r->stopHead, *nxt;
    while (cur) { nxt = cur->next; cur->next = prev; prev = cur; cur = nxt; }
    r->stopHead = prev;
    printf("Evening: ");
    for (StopNode *t = r->stopHead; t; t = t->next)
        printf("%s%s", t->name, t->next ? " -> " : "");
    printf("\n");
}

void undoFee(void) {
    if (!top) { printf("Nothing to undo.\n"); return; }
    top->stop->fee = top->oldFee;
    printf("Fee restored: %s Rs. %.2f\n", top->stop->name, top->oldFee);
    FeeLog *old = top;
    top = top->below;                                 /* pop */
    free(old);
}

void insertBetween(void) {
    char after[LEN], name[LEN];
    ask("Insert after which stop? ", after);
    StopNode *t = search(after);
    if (!t) { printf("Stop not found.\n"); return; }
    ask("New stop name: ", name);
    if (name[0] == '\0' || search(name)) { printf("Invalid or duplicate stop.\n"); return; }
    double fee = askFee("Annual fee (Rs.): ");
    if (fee < 0) return;
    StopNode *n = (StopNode *)malloc(sizeof(StopNode));
    strncpy(n->name, name, LEN - 1); n->name[LEN - 1] = '\0';
    n->fee = fee;
    n->next = t->next;                                /* step 1 */
    t->next = n;                                      /* step 2 */
    printf("Stop inserted.\n");
}

void compareStops(void) {
    char a[LEN], b[LEN];
    ask("First stop: ", a);
    ask("Second stop: ", b);
    StopNode *x = search(a), *y = search(b);
    if (!x || !y) { printf("One or both stops not found.\n"); return; }
    printf("%s Rs. %.0f vs %s Rs. %.0f\n", x->name, x->fee, y->name, y->fee);
    printf("Difference: Rs. %.2f\n", fabs(x->fee - y->fee));
}

/* ---------- file load / save ---------- */
void seed(void) {
    RouteNode *r = addRoute("Route 1 - Trichy Central");
    addStop(r, "Central Bus Stand", 18000); addStop(r, "Thillai Nagar", 19500); addStop(r, "Woraiyur", 21000);
    r = addRoute("Route 2 - Srirangam");
    addStop(r, "Srirangam", 22000); addStop(r, "Thiruvanaikaval", 23000); addStop(r, "Kallanai Road", 24500);
    r = addRoute("Route 3 - Lalgudi");
    addStop(r, "Samayapuram", 25000); addStop(r, "Manachanallur", 27000); addStop(r, "Lalgudi", 29000);
}

void load(void) {
    FILE *f = fopen(FILE_NAME, "r");
    if (!f) { seed(); return; }
    char line[300];
    while (fgets(line, sizeof line, f)) {
        line[strcspn(line, "\r\n")] = '\0';
        if (line[0] == '\0') continue;
        char *route = strtok(line, "|");
        char *stop = strtok(NULL, "|");
        char *fee = strtok(NULL, "|");
        if (!route) continue;
        RouteNode *r = findRoute(route);
        if (!r) r = addRoute(route);
        if (stop && fee) addStop(r, stop, atof(fee));
    }
    fclose(f);
}

void save(void) {
    FILE *f = fopen(FILE_NAME, "w");
    if (!f) { printf("Could not save data file.\n"); return; }
    for (RouteNode *r = head; r; r = r->next) {
        if (!r->stopHead) fprintf(f, "%s\n", r->routeName);
        for (StopNode *t = r->stopHead; t; t = t->next)
            fprintf(f, "%s|%s|%.2f\n", r->routeName, t->name, t->fee);
    }
    fclose(f);
}

/* ---------- main ---------- */
int main(void) {
    char c[LEN];
    load();
    while (1) {
        printf("\n==============================\n");
        printf("  BUS ROUTE FARE MANAGEMENT\n");
        printf("==============================\n");
        printf("1. Add Route / Stop\n2. Display Routes\n3. Search Stop\n4. Update Fee\n");
        printf("5. Delete Stop\n6. Count Stops\n7. Highest Fee Stop\n8. Reverse Route (Evening)\n");
        printf("9. Undo Last Fee Change\n10. Insert Between Stops\n11. Compare Two Stops\n12. Exit\n");
        printf("Enter your choice: ");
        if (!fgets(c, LEN, stdin)) { save(); return 0; }
        c[strcspn(c, "\r\n")] = '\0';
        int ch = atoi(c);
        switch (ch) {
            case 1: addRouteOrStop(); break;
            case 2: display(); break;
            case 3: searchStop(); break;
            case 4: updateFee(); break;
            case 5: deleteStop(); break;
            case 6: countStops(); break;
            case 7: highestFee(); break;
            case 8: reverseRoute(); break;
            case 9: undoFee(); break;
            case 10: insertBetween(); break;
            case 11: compareStops(); break;
            case 12: save(); printf("Data saved. Goodbye!\n"); return 0;
            default: printf("Invalid choice. Enter 1-12.\n");
        }
    }
}
