# SOP1 L1 – praktyczny podręcznik: system plików i C

Oct 19, 2026 · @Kinga

Ten podręcznik zbiera w jednym miejscu wszystko, czego potrzebujesz na L1 z SOP1: szablon programu, pamięć i napisy w C, funkcje z `<string.h>`, strumienie `FILE*`, katalogi, `stat`, uprawnienia, dowiązania, `nftw`, deskryptory i gotowe przepisy z rozwiązanych zadań. Każda funkcja ma: nagłówek, sygnaturę, co znaczą argumenty, co zwraca, jak sprawdzić błąd i krótki przykład.

## Spis treści

1. Szablon programu na laby
2. Pamięć i napisy w C od podstaw
3. Funkcje tekstowe `<string.h>`
4. Formatowanie i konwersje liczb
5. Argumenty programu i `getopt`
6. Obsługa błędów: `errno`, `perror`, `ERR`
7. Strumienie `FILE*` (`fopen`, `fgets`, `getline`, `fread`…)
8. Katalogi: `opendir`, `readdir`, `closedir`
9. Informacje o pliku: `stat`, `lstat`, typy i uprawnienia
10. Katalog roboczy i ścieżki: `getcwd`, `chdir`, łączenie ścieżek
11. Tworzenie, usuwanie, uprawnienia: `mkdir`, `unlink`, `rename`, `chmod`, `access`, `umask`
12. Dowiązania: `symlink`, `readlink`, `link`
13. Rekurencyjne przeglądanie: `nftw`
14. Niskopoziomowe I/O: `open`, `read`, `write`, `lseek`, `readv`
15. Środowisko, losowość, czas: `getenv`, `rand`, `time`
16. Gotowe przepisy (kod do skopiowania)
17. Odpowiedzi na pytania z tutorialu
18. Pułapki, checklista, komendy terminala i strony `man`

**Jak czytać zapis funkcji.** `const char *` = funkcja tylko czyta napis. `char *` = może do niego pisać. `restrict` = argumenty nie mogą wskazywać na tę samą pamięć. Wartość zwracana „0 / -1” oznacza: 0 sukces, -1 błąd i ustawione `errno`. Strony podręcznika otwierasz przez `man 3p nazwa` (wersja POSIX, ta obowiązuje na labach).

## 1. Szablon programu na laby

Zacznij każde zadanie od tego szkieletu: ma komplet nagłówków, makro `ERR`, `usage` i pętlę `getopt`.

```c
#define _XOPEN_SOURCE 700   // MUSI być przed #include: włącza POSIX 2008 (getline, strdup, nftw, readlink...)

#include <dirent.h>     // opendir, readdir, closedir, DIR, struct dirent
#include <errno.h>      // errno, ENOENT, EEXIST...
#include <fcntl.h>      // open, O_RDONLY, O_CREAT...
#include <ftw.h>        // nftw, FTW_PHYS, FTW_F, FTW_D
#include <limits.h>     // PATH_MAX
#include <stdio.h>      // printf, fopen, fgets, getline, perror...
#include <stdlib.h>     // malloc, free, exit, strtol, getenv, rand
#include <string.h>     // strlen, strcpy, strcmp, strchr, strdup...
#include <sys/stat.h>   // stat, lstat, mkdir, chmod, umask, S_ISDIR...
#include <sys/uio.h>    // readv, writev, struct iovec
#include <time.h>       // time
#include <unistd.h>     // getopt, chdir, getcwd, unlink, symlink, read, write, close

#define ERR(source) (perror(source), fprintf(stderr, "%s:%d\n", __FILE__, __LINE__), exit(EXIT_FAILURE))

void usage(char *pname)
{
    fprintf(stderr, "USAGE: %s -p <path> [-o <file>]\n", pname);
    exit(EXIT_FAILURE);
}

int main(int argc, char **argv)
{
    int c;
    while ((c = getopt(argc, argv, "p:o:")) != -1)
        switch (c)
        {
            case 'p': /* optarg = ścieżka */ break;
            case 'o': /* optarg = plik */   break;
            default: usage(argv[0]);
        }
    if (optind < argc)   // nadmiarowe argumenty pozycyjne
        usage(argv[0]);
    return EXIT_SUCCESS;
}
```

**Kompilacja i sprawdzanie.**

```bash
gcc -Wall -Wextra -o prog prog.c                          # zwykła kompilacja, poprawiaj KAŻDE ostrzeżenie
gcc -Wall -Wextra -g -fsanitize=address,undefined -o prog prog.c   # wykrywa wycieki i błędy pamięci
valgrind --leak-check=full ./prog -p .                     # alternatywa dla sanitizera
```

- `_XOPEN_SOURCE 700` musi być pierwszą linią. Bez niego `getline`, `strdup` czy `nftw` mogą być niewidoczne i dostaniesz „implicit declaration”.
- `EXIT_SUCCESS` (0) i `EXIT_FAILURE` (1) to kody wyjścia z `<stdlib.h>`. Sprawdzisz je w terminalu przez `echo $?`.
- Kod rozbijaj na krótkie funkcje: dane wejściowe przez parametry, wynik przez wartość zwracaną lub wskaźnik, bez zmiennych globalnych.

## 2. Pamięć i napisy w C od podstaw

Napis w C to ciąg bajtów `char` zakończony bajtem zerowym `'\0'`. Nie ma typu „string”: każda funkcja szuka końca napisu po tym zerze, więc bufor musi mieć zawsze o 1 bajt więcej niż liczba znaków.

### 2.1 Tablica a wskaźnik

```c
char a[] = "kot";       // tablica 4 bajtów na stosie: 'k','o','t','\0'; można zmieniać: a[0] = 'K';
char b[10] = "kot";     // 10 bajtów, reszta wyzerowana
const char *c = "kot";  // wskaźnik na stały napis w pamięci programu: c[0] = 'K' -> SEGFAULT
char *d;                 // wskaźnik NIEZAINICJOWANY: nie wskazuje na żadną pamięć!

sizeof(a)   // 4  - rozmiar tablicy w bajtach (z '\0')
strlen(a)   // 3  - liczba znaków do '\0'
sizeof(c)   // 8  - rozmiar WSKAŹNIKA, nie napisu! (typowy błąd)
```

- `char *` to tylko adres. Zanim coś pod niego zapiszesz, musi wskazywać na tablicę albo pamięć z `malloc`.
- Tablicy nie da się przypisać (`a = "pies";` nie skompiluje się). Kopiujesz przez `strcpy` / `snprintf`.
- Napisów nie porównujesz przez `==` (to porównuje adresy). Użyj `strcmp`.
- Tablica przekazana do funkcji zamienia się we wskaźnik, więc w funkcji `sizeof` nie poda jej rozmiaru. Przekaż rozmiar osobnym argumentem.

### 2.2 Pamięć dynamiczna `<stdlib.h>`

| Funkcja | Sygnatura | Co robi | Zwraca |
| --- | --- | --- | --- |
| `malloc` | `void *malloc(size_t size)` | rezerwuje `size` bajtów, zawartość przypadkowa | wskaźnik albo `NULL` |
| `calloc` | `void *calloc(size_t n, size_t size)` | rezerwuje `n*size` bajtów i zeruje je | wskaźnik albo `NULL` |
| `realloc` | `void *realloc(void *ptr, size_t size)` | zmienia rozmiar bloku, kopiuje starą zawartość | nowy wskaźnik albo `NULL` (stary blok wtedy nadal ważny) |
| `free` | `void free(void *ptr)` | zwalnia blok z `malloc`/`calloc`/`realloc`/`strdup`/`getline` | nic; `free(NULL)` jest bezpieczne |

```c
// napis na n znaków: n + 1 bajtów na '\0'
char *s = malloc(n + 1);
if (!s)
    ERR("malloc");

// tablica 100 int-ów, wyzerowana
int *t = calloc(100, sizeof(int));
if (!t)
    ERR("calloc");

// powiększanie: NIGDY p = realloc(p, ...) - przy błędzie zgubisz stary blok
int *tmp = realloc(t, 200 * sizeof(int));
if (!tmp)
    ERR("realloc");
t = tmp;

// tablica napisów (np. lista ścieżek z -p)
char **paths = malloc(argc * sizeof(char *));

free(s);
free(t);
free(paths);
s = NULL;   // dobra praktyka: po free wskaźnik na NULL, żeby nie użyć go przez pomyłkę
```

**Zasady.**

- Każdy `malloc` sprawdzaj (`if (!p) ERR("malloc");`) i każdy kiedyś zwolnij jednym `free`. Na labach za wycieki odejmują punkty.
- Mnoż przez `sizeof(typ)`: `malloc(n * sizeof(int))`, nie `malloc(n)`.
- Nie zwalniaj dwa razy tego samego bloku i nie używaj pamięci po `free`.
- Nie zwalniaj pamięci, której nie alokowałeś: `argv[i]`, `optarg`, wyniku `getenv`, `readdir`, literałów napisowych.
- Funkcje, które zwracają nową pamięć do zwolnienia: `malloc`, `calloc`, `realloc`, `strdup`, `strndup`, bufor z `getline`.

## 3. Funkcje tekstowe `<string.h>`

Wszystkie poniższe funkcje zakładają napisy zakończone `'\0'` (oprócz `mem*`, które działają na bajtach o podanej długości). Żadna z nich nie sprawdza rozmiaru bufora docelowego: to Twoje zadanie.

### 3.1 Szybka ściąga

| Funkcja | Sygnatura | Co zwraca |
| --- | --- | --- |
| `strlen` | `size_t strlen(const char *s)` | liczbę znaków bez `'\0'` |
| `strcpy` | `char *strcpy(char *dst, const char *src)` | `dst` |
| `strncpy` | `char *strncpy(char *dst, const char *src, size_t n)` | `dst` (może NIE dodać `'\0'`!) |
| `strcat` | `char *strcat(char *dst, const char *src)` | `dst` |
| `strncat` | `char *strncat(char *dst, const char *src, size_t n)` | `dst` (zawsze dodaje `'\0'`) |
| `strcmp` | `int strcmp(const char *a, const char *b)` | 0 równe, <0 gdy a\<b, >0 gdy a>b |
| `strncmp` | `int strncmp(const char *a, const char *b, size_t n)` | jak `strcmp`, tylko pierwsze `n` znaków |
| `strchr` | `char *strchr(const char *s, int c)` | wskaźnik na PIERWSZE `c` albo `NULL` |
| `strrchr` | `char *strrchr(const char *s, int c)` | wskaźnik na OSTATNIE `c` albo `NULL` |
| `strstr` | `char *strstr(const char *s, const char *sub)` | wskaźnik na początek `sub` w `s` albo `NULL` |
| `strdup` | `char *strdup(const char *s)` | NOWĄ kopię (do `free`) albo `NULL` |
| `strndup` | `char *strndup(const char *s, size_t n)` | nową kopię max `n` znaków + `'\0'` albo `NULL` |
| `strtok` | `char *strtok(char *s, const char *delim)` | kolejny token albo `NULL` (niszczy `s`) |
| `strspn` / `strcspn` | `size_t strcspn(const char *s, const char *reject)` | długość początku bez znaków z `reject` |
| `memset` | `void *memset(void *p, int c, size_t n)` | `p` |
| `memcpy` | `void *memcpy(void *dst, const void *src, size_t n)` | `dst` (obszary NIE mogą się nakładać) |
| `memmove` | `void *memmove(void *dst, const void *src, size_t n)` | `dst` (obszary mogą się nakładać) |
| `memcmp` | `int memcmp(const void *a, const void *b, size_t n)` | jak `strcmp`, na `n` bajtach |
| `strerror` | `char *strerror(int errnum)` | opis błędu, np. „No such file or directory” |

### 3.2 Długość i kopiowanie

```c
char buf[64];
const char *src = "library/a/b";

size_t len = strlen(src);          // 11

// strcpy: dst musi mieć >= strlen(src) + 1 bajtów
if (len + 1 > sizeof(buf))
    ERR("too long");
strcpy(buf, src);                  // buf = "library/a/b"

// strncpy: kopiuje max n bajtów; jeśli src dłuższy, NIE wstawi '\0'
strncpy(buf, src, sizeof(buf) - 1);
buf[sizeof(buf) - 1] = '\0';       // zawsze dopisz zero ręcznie

// najbezpieczniej: snprintf (sam pilnuje rozmiaru i '\0')
snprintf(buf, sizeof(buf), "%s", src);
```

### 3.3 Sklejanie

```c
char path[PATH_MAX];
strcpy(path, "index");
strcat(path, "/");                 // path = "index/"
strcat(path, "by-title");          // path = "index/by-title"

// to samo w jednej linii, bezpiecznie:
snprintf(path, sizeof(path), "%s/%s", "index", "by-title");

// sklejanie do NOWEJ pamięci (rozmiar = suma długości + '/' + '\0')
char *res = malloc(strlen(a) + strlen(b) + 2);
if (!res)
    ERR("malloc");
strcpy(res, a);
strcat(res, "/");
strcat(res, b);
// ... użyj res ...
free(res);
```

### 3.4 Porównywanie

```c
if (strcmp(name, ".") == 0 || strcmp(name, "..") == 0)  // pomiń . i ..
    continue;

if (!strcmp(key, "author"))        // !strcmp(...) == "są równe"
    ...

// czy line zaczyna się od "numpy " ?
size_t n = strlen(pkg);
if (strncmp(line, pkg, n) == 0 && line[n] == ' ')
    ...
```

`strcmp` NIE zwraca prawdy/fałszu. Zwraca 0 dla równych napisów, więc `if (strcmp(a, b))` znaczy „są RÓŻNE”.

### 3.5 Szukanie znaków i podnapisów

```c
// klucz:wartość - rozcięcie napisu w miejscu ':'
char line[] = "title:Odyssey";
char *sep = strchr(line, ':');     // wskaźnik na ':' albo NULL
if (sep) {
    *sep = '\0';                   // line = "title"
    char *value = sep + 1;         // value = "Odyssey"
}

// rozszerzenie pliku: ostatnia kropka
const char *dot = strrchr("arch.tar.gz", '.');   // ".gz"
if (dot && strcmp(dot + 1, "gz") == 0) ...

// nazwa pliku ze ścieżki: ostatni '/'
const char *slash = strrchr(path, '/');
const char *fname = slash ? slash + 1 : path;

// "numpy==1.0.0" -> nazwa i wersja
char spec[] = "numpy==1.0.0";
char *eq = strstr(spec, "==");
if (eq) {
    *eq = '\0';                    // spec = "numpy"
    char *version = eq + 2;        // "1.0.0"
}

// obcięcie '\n' z końca linii
line[strcspn(line, "\n")] = '\0';
```

`sep - line` daje pozycję (indeks) znalezionego znaku: odejmowanie wskaźników w tej samej tablicy.

### 3.6 Kopie w nowej pamięci: `strdup`, `strndup`

```c
char *copy = strdup(value);        // malloc(strlen(value)+1) + strcpy
if (!copy)
    ERR("strdup");

char *title64 = strndup(title, 64); // max 64 bajty + '\0' (przycinanie tytułu)
if (!title64)
    ERR("strndup");

free(copy);
free(title64);
```

Użyj `strdup`, gdy napis ma przeżyć bufor, z którego pochodzi (np. bufor `getline` jest nadpisywany w następnej iteracji).

### 3.7 Dzielenie na tokeny: `strtok`

```c
char s[] = "numpy 1.0.0";          // MUSI być modyfikowalne (tablica, nie literał)
char *name = strtok(s, " ");       // pierwszy token: "numpy"
char *ver  = strtok(NULL, " ");    // kolejne: NULL zamiast s -> "1.0.0"
char *x    = strtok(NULL, " ");    // NULL - koniec
```

`strtok` wstawia `'\0'` w miejsca separatorów i pamięta stan globalnie. Nie używaj go w dwóch pętlach naraz; wówczas jest `strtok_r`.

### 3.8 Operacje na bajtach: `memset`, `memcpy`, `memcmp`

```c
char title[65];
memset(title, 0, sizeof(title));   // wyzeruj cały bufor

struct stat st;
memset(&st, 0, sizeof(st));        // wyzeruj strukturę

char rec[68];
unsigned int size = 1234;
memcpy(rec, &size, 4);             // 4 bajty liczby do bufora
memcpy(rec + 4, title, 64);        // 64 bajty tytułu za nią

if (memcmp(a, b, 64) == 0) ...     // porównanie 64 bajtów (też z zerami w środku)
```

## 4. Formatowanie i konwersje liczb

Do zamiany liczb na tekst służą `printf`/`fprintf`/`snprintf`, a do zamiany tekstu na liczby `strtol` (zawsze lepszy niż `atoi`, bo wykrywa błędy).

### 4.1 Rodzina `printf` `<stdio.h>`

| Funkcja | Sygnatura | Dokąd pisze | Zwraca |
| --- | --- | --- | --- |
| `printf` | `int printf(const char *fmt, ...)` | stdout | liczbę znaków albo <0 |
| `fprintf` | `int fprintf(FILE *f, const char *fmt, ...)` | strumień `f` (np. `stderr`, plik) | liczbę znaków albo <0 |
| `snprintf` | `int snprintf(char *buf, size_t n, const char *fmt, ...)` | do bufora, max `n-1` znaków + `'\0'` | ile znaków CHCIAŁ zapisać (>= n oznacza obcięcie) |
| `sprintf` | `int sprintf(char *buf, const char *fmt, ...)` | do bufora BEZ limitu | unikaj, grozi przepełnieniem |

| Specyfikator | Typ | Przykład |
| --- | --- | --- |
| `%d` / `%i` | `int` | `printf("%d", 42)` |
| `%u` | `unsigned int` | rozmiar z bazy binarnej |
| `%ld` / `%lu` | `long` / `unsigned long` |  |
| `%lld` | `long long` | `(long long)st.st_size` |
| `%zu` / `%zd` | `size_t` / `ssize_t` | `strlen(s)`, wynik `read` |
| `%o` | liczba ósemkowo | `printf("%o", st.st_mode & 0777)` daje np. `644` |
| `%x` | liczba szesnastkowo |  |
| `%c` | znak | `'A' + i` |
| `%s` | napis `char *` |  |
| `%.*s` | napis o podanej długości | `printf("%.*s", 64, title)` |
| `%5d` / `%-10s` | szerokość pola / wyrównanie do lewej |  |
| `%%` | znak `%` |  |

```c
fprintf(stderr, "cannot open %s: %s\n", path, strerror(errno));
printf("%s %lld\n", entry->d_name, (long long)st.st_size);   // off_t rzutuj na long long

// bezpieczne sklejanie ścieżki z kontrolą obcięcia
char full[PATH_MAX];
int n = snprintf(full, sizeof(full), "%s/%s", dir, name);
if (n < 0 || n >= (int)sizeof(full))
    ERR("snprintf");
```

Nigdy nie pisz `printf(argv[i])` ani `printf(napis_od_użytkownika)`. Gdy w napisie będzie `%d`, printf odczyta śmieci ze stosu. Zawsze `printf("%s", argv[i])`.

### 4.2 Wczytywanie z formatem: `scanf`, `fscanf`, `sscanf`

```c
int a, b;
if (sscanf("12 34", "%d %d", &a, &b) != 2)   // zwraca liczbę wczytanych pól
    ERR("sscanf");

char key[64], val[256];
// %63[^:] = do 63 znaków różnych od ':'
if (sscanf(line, "%63[^:]:%255[^\n]", key, val) == 2) ...
```

Podawaj w `%s` / `%[...]` maksymalną szerokość (bufor - 1), inaczej grozi przepełnienie.

### 4.3 Tekst na liczbę: `strtol`, `strtoul`, `atoi`

```c
long strtol(const char *s, char **endptr, int base);
unsigned long strtoul(const char *s, char **endptr, int base);
int atoi(const char *s);   // brak wykrywania błędów: "abc" -> 0
```

- `s`: napis z liczbą. Początkowe spacje są pomijane.
- `endptr`: tu funkcja wpisze adres pierwszego znaku, który nie był częścią liczby. Może być `NULL`.
- `base`: 10 dziesiętnie, 8 ósemkowo (uprawnienia `"755"`), 16 szesnastkowo, 0 = wykryj z prefiksu (`0x`, `0`).
- Zwraca liczbę. Przy przekroczeniu zakresu zwraca `LONG_MAX`/`LONG_MIN` i ustawia `errno = ERANGE`.

```c
// pełna walidacja argumentu -d <depth>
char *end;
errno = 0;
long depth = strtol(optarg, &end, 10);
if (errno || end == optarg || *end != '\0' || depth < 1)
    usage(argv[0]);      // "", "abc", "5x", "0" -> błąd

// uprawnienia podane ósemkowo: -p 644
mode_t perms = strtol(optarg, NULL, 8);   // "644" -> 0644 = 420 dziesiętnie

// rozmiar
ssize_t size = strtol(optarg, NULL, 10);
```

### 4.4 Klasyfikacja znaków `<ctype.h>`

| Funkcja | Prawda gdy znak to |
| --- | --- |
| `isdigit(c)` | cyfra 0-9 |
| `isalpha(c)` | litera |
| `isalnum(c)` | litera lub cyfra |
| `isspace(c)` | spacja, `\t`, `\n`, `\r`... |
| `isupper(c)` / `islower(c)` | wielka / mała litera |
| `toupper(c)` / `tolower(c)` | (zwraca zamieniony znak) |

```c
#include <ctype.h>
for (char *p = s; *p; p++)
    *p = toupper((unsigned char)*p);   // rzut na unsigned char: wymagany dla znaków > 127 (UTF-8)

// usunięcie białych znaków z końca
size_t n = strlen(s);
while (n > 0 && isspace((unsigned char)s[n - 1]))
    s[--n] = '\0';

// losowa wielka litera
char c = 'A' + rand() % ('Z' - 'A' + 1);   // 26 liter
```

## 5. Argumenty programu i `getopt`

`getopt` czyta opcje w stylu `-p x -o y` w dowolnej kolejności i jest standardowym sposobem na labach. Parametry pozycyjne (bez myślnika) zostają na końcu `argv` od indeksu `optind`.

### 5.1 `argc` i `argv`

```c
int main(int argc, char **argv)
// ./prog -p dir1 plik
// argc = 4
// argv[0] = "./prog", argv[1] = "-p", argv[2] = "dir1", argv[3] = "plik", argv[4] = NULL

for (int i = 1; i < argc; i++)     // wszystkie parametry pozycyjne (bez getopt)
    printf("%s\n", argv[i]);
```

### 5.2 `getopt` `<unistd.h>`

```c
int getopt(int argc, char *const argv[], const char *optstring);
extern char *optarg;   // argument bieżącej opcji (dla "p:")
extern int optind;     // indeks pierwszego nieprzetworzonego argumentu
extern int opterr;     // 0 = getopt nie wypisuje własnych błędów
extern int optopt;     // znak nieznanej opcji
```

- `optstring`: litery opcji. Dwukropek po literze (`"p:"`) = opcja wymaga argumentu. Bez dwukropka (`"c"`) = flaga.
- Zwraca: literę opcji, `'?'` dla nieznanej opcji lub brakującego argumentu, `-1` gdy opcje się skończyły.
- `optarg` wskazuje na tekst w `argv`: nie zwalniaj go i nie musisz kopiować.
- W Linuksie `getopt` przestawia argumenty, więc `./prog plik -p x` też zadziała; pozycyjne trafią na koniec.

### 5.3 Wzorzec: wielokrotne `-p`, jednokrotne `-o`, flaga `-c`, liczba `-d`

```c
int main(int argc, char **argv)
{
    char **paths = malloc(argc * sizeof(char *));  // -p nie wystąpi więcej niż argc razy
    if (!paths)
        ERR("malloc");
    int path_count = 0;
    char *out_name = NULL;
    int create = 0;
    long depth = 1;
    int c;

    while ((c = getopt(argc, argv, "p:o:cd:")) != -1)
    {
        switch (c)
        {
            case 'p':
                paths[path_count++] = optarg;
                break;
            case 'o':
                if (out_name)           // drugie -o = błąd
                    usage(argv[0]);
                out_name = optarg;
                break;
            case 'c':
                create = 1;
                break;
            case 'd':
            {
                char *end;
                depth = strtol(optarg, &end, 10);
                if (*end || depth < 1)
                    usage(argv[0]);
                break;
            }
            default:                    // '?': nieznana opcja / brak argumentu
                usage(argv[0]);
        }
    }
    if (optind < argc)                  // nadmiarowe parametry pozycyjne
        usage(argv[0]);
    if (path_count == 0)                // wymagane -p
        usage(argv[0]);

    for (int i = 0; i < path_count; i++)
        process(paths[i]);

    free(paths);
    return EXIT_SUCCESS;
}
```

**Parametr pozycyjny po opcjach** (np. `./prog -c nazwa`): po pętli `getopt` jest w `argv[optind]`.

```c
if (optind == argc - 1)       // dokładnie jeden pozycyjny
    name = argv[optind];
```

**Ignorowanie nieznanych opcji**: jeśli zadanie mówi, żeby ignorować nieznane opcje, ustaw `opterr = 0;` przed pętlą i w `default:` daj `break;` zamiast `usage`.

## 6. Obsługa błędów: `errno`, `perror`, `ERR`

Sprawdzaj wartość zwracaną KAŻDEJ funkcji systemowej i bibliotecznej, która może zawieść. Wzorzec na labach to `if (funkcja(...)) ERR("funkcja");`.

### 6.1 Jak funkcje zgłaszają błąd

| Rodzaj funkcji | Sukces | Błąd | Przykłady |
| --- | --- | --- | --- |
| zwraca `int` status | `0` | `-1` (lub ≠ 0) | `stat`, `lstat`, `mkdir`, `chdir`, `unlink`, `rename`, `chmod`, `symlink`, `closedir`, `close`, `fclose` (EOF) |
| zwraca wskaźnik | wskaźnik | `NULL` | `fopen`, `opendir`, `malloc`, `strdup`, `getcwd` |
| zwraca deskryptor | `>= 0` | `-1` | `open` |
| zwraca liczbę bajtów | `>= 0` | `-1` | `read`, `write`, `readv`, `readlink`, `getline` |
| `NULL` znaczy też „koniec” | wskaźnik | `NULL` + `errno ≠ 0` | `readdir` (wyzeruj `errno` przed!) |
| nie zwraca błędów |  |  | `umask`, `getpid`, `free` |

### 6.2 `errno` `<errno.h>`

Globalna zmienna `int errno` z kodem ostatniego błędu. Funkcje ustawiają ją przy błędzie, ale przy sukcesie nie muszą jej zerować, więc sprawdzaj ją wyłącznie wtedy, gdy funkcja zwróciła błąd. Każde kolejne wywołanie (nawet `printf`) może ją nadpisać.

| Kod | Znaczenie | Kiedy typowo |
| --- | --- | --- |
| `ENOENT` | No such file or directory | plik / katalog nie istnieje |
| `EEXIST` | File exists | `mkdir`, `open(O_EXCL)`, `symlink` na istniejącej nazwie |
| `EACCES` | Permission denied | brak praw |
| `ENOTDIR` | Not a directory | element ścieżki nie jest katalogiem |
| `EISDIR` | Is a directory | `fopen` katalogu do zapisu |
| `ENOTEMPTY` | Directory not empty | `rmdir` niepustego |
| `EINVAL` | Invalid argument | zły argument |
| `ERANGE` | Result too large | `strtol` poza zakresem, `getcwd` za mały bufor |
| `EINTR` | Interrupted system call | sygnał przerwał funkcję (L2+) |

### 6.3 `perror` i `strerror`

```c
void perror(const char *s);      // <stdio.h>: wypisuje na stderr "s: opis errno"
char *strerror(int errnum);      // <string.h>: zwraca opis jako napis (nie zwalniaj)

perror("opendir");                              // opendir: No such file or directory
fprintf(stderr, "%s: %s\n", path, strerror(errno));  // własny format z nazwą pliku
```

### 6.4 Makro `ERR`

```c
#define ERR(source) (perror(source), fprintf(stderr, "%s:%d\n", __FILE__, __LINE__), exit(EXIT_FAILURE))
```

Wypisuje opis błędu, plik i linię w kodzie, po czym kończy program kodem 1. Używaj go, gdy dalsza praca nie ma sensu.

### 6.5 Kiedy kończyć, a kiedy kontynuować

```c
// 1) błąd krytyczny -> koniec
if (!(f = fopen(name, "r")))
    ERR("fopen");

// 2) błąd oczekiwany -> ignoruj konkretny kod
if (unlink(name) && errno != ENOENT)     // brak pliku to nie problem
    ERR("unlink");
if (mkdir(dir, 0755) && errno != EEXIST) // katalog już jest - OK
    ERR("mkdir");

// 3) błąd jednego elementu -> komunikat i następny element
for (int i = 0; i < n; i++) {
    DIR *d = opendir(paths[i]);
    if (!d) {
        fprintf(stderr, "cannot open %s: %s\n", paths[i], strerror(errno));
        continue;
    }
    ...
}

// 4) błąd użytkownika (składnia) -> komunikat + usage
if (!name)
    usage(argv[0]);

// 5) własny komunikat, ale errno potrzebne później -> zapamiętaj je
int saved = errno;
fprintf(stderr, "duplicate: %s\n", name);
errno = saved;
ERR("symlink");
```

## 7. Strumienie `FILE*` `<stdio.h>`

Wysokopoziomowe API: otwierasz plik `fopen`, czytasz/piszesz funkcjami `f*`, zamykasz `fclose`. Biblioteka buforuje dane, więc jest wygodna i szybka do tekstu.

### 7.1 `fopen` i `fclose`

```c
FILE *fopen(const char *restrict pathname, const char *restrict mode);
int fclose(FILE *stream);    // 0 sukces, EOF błąd
```

| Tryb | Czytanie | Pisanie | Plik nie istnieje | Plik istnieje | Kursor na starcie |
| --- | --- | --- | --- | --- | --- |
| `"r"` | tak | nie | błąd `ENOENT` | bez zmian | początek |
| `"r+"` | tak | tak | błąd | bez zmian | początek |
| `"w"` | nie | tak | tworzy | OBCINA do 0 | początek |
| `"w+"` | tak | tak | tworzy | OBCINA do 0 | początek |
| `"a"` | nie | tak | tworzy | bez zmian | ZAWSZE koniec (`fseek` nie pomaga) |
| `"a+"` | tak | tak | tworzy | bez zmian | czytanie od początku, pisanie na końcu |

- Dopisek `b` (`"rb"`) w POSIX nic nie zmienia.
- `fopen` tworzy pliki z prawami `0666 & ~umask`. Nie da się nadać `x`. Gdy potrzebujesz konkretnych praw, użyj `umask` albo `open` (rozdział 14).
- Jeśli plik istniał, `"w"` zachowuje jego stare uprawnienia. Dlatego w zadaniach najpierw `unlink`.

```c
FILE *f = fopen(path, "r");
if (!f)
    ERR("fopen");
/* ... */
if (fclose(f))
    ERR("fclose");
```

Gotowe strumienie: `stdin`, `stdout`, `stderr`. Możesz trzymać `FILE *out = stdout;` i podmienić na plik, gdy jest opcja `-o`. Nie zamykaj wtedy `stdout`: `if (out != stdout) fclose(out);`.

### 7.2 Pisanie

| Funkcja | Sygnatura | Zwraca przy błędzie |
| --- | --- | --- |
| `fprintf` | `int fprintf(FILE *f, const char *fmt, ...)` | wartość < 0 |
| `fputs` | `int fputs(const char *s, FILE *f)` (nie dodaje `\n`) | `EOF` |
| `fputc` | `int fputc(int c, FILE *f)` | `EOF` |
| `fwrite` | `size_t fwrite(const void *p, size_t size, size_t n, FILE *f)` | mniej niż `n` |
| `puts` | `int puts(const char *s)` (na stdout, DODAJE `\n`) | `EOF` |

```c
fprintf(out, "%s %s\n", name, version);
fputs(line, out);                     // przepisanie linii z getline (ma już '\n')
fputc('A' + rand() % 26, f);
if (fwrite(buf, 1, len, f) != len)    // dane binarne
    ERR("fwrite");
```

### 7.3 Czytanie linia po linii: `getline` (POLECANE)

```c
ssize_t getline(char **restrict lineptr, size_t *restrict n, FILE *restrict stream);
```

- `*lineptr`: bufor. Zacznij od `NULL` i `n = 0`, a `getline` sam go zaalokuje i będzie powiększał.
- Zwraca długość wczytanej linii (z `'\n'`, jeśli był), albo `-1` przy końcu pliku lub błędzie.
- Po pętli zwolnij bufor jednym `free(line)`.

```c
char *line = NULL;
size_t cap = 0;
ssize_t len;
while ((len = getline(&line, &cap, f)) != -1)
{
    if (len > 0 && line[len - 1] == '\n')   // utnij '\n' (ostatnia linia może go nie mieć)
        line[--len] = '\0';
    printf("[%s]\n", line);
}
if (ferror(f))                              // -1 było błędem, a nie końcem pliku?
    ERR("getline");
free(line);
```

### 7.4 Inne sposoby czytania

```c
char *fgets(char *restrict s, int n, FILE *restrict f);   // max n-1 znaków + '\0', zachowuje '\n'; NULL = koniec/błąd
int fgetc(FILE *f);                                       // znak jako int albo EOF
size_t fread(void *restrict p, size_t size, size_t n, FILE *restrict f);  // liczba ELEMENTÓW
int fscanf(FILE *restrict f, const char *restrict fmt, ...);

char buf[256];
while (fgets(buf, sizeof(buf), f))          // linie dłuższe niż 255 zostaną pocięte!
    fputs(buf, stdout);

int ch;                                     // int, nie char - inaczej nie odróżnisz EOF
while ((ch = fgetc(f)) != EOF)
    putchar(ch);

int tab[100];
size_t got = fread(tab, sizeof(int), 100, f);   // np. 37 gdy plik się skończył
```

**Doczytywanie po niepełnym `fread`**: jeśli zwrócił `k < n`, wywołaj ponownie z `ptr + k*size` i `n - k`. Gdy `feof(f)`, danych już nie ma.

### 7.5 Pozycja w pliku: `fseek`, `ftell`, `rewind`

```c
int fseek(FILE *f, long offset, int whence);   // 0 sukces, -1 błąd
long ftell(FILE *f);                            // pozycja albo -1
void rewind(FILE *f);                           // = fseek(f, 0, SEEK_SET)
```

| `whence` | Liczone od | Przykład |
| --- | --- | --- |
| `SEEK_SET` | początku pliku | `fseek(f, 100, SEEK_SET)`: bajt nr 100 |
| `SEEK_CUR` | bieżącej pozycji | `fseek(f, -4, SEEK_CUR)`: cofnij o 4 |
| `SEEK_END` | końca pliku | `fseek(f, 0, SEEK_END)`: za ostatnim bajtem |

```c
// rozmiar pliku
if (fseek(f, 0, SEEK_END)) ERR("fseek");
long size = ftell(f);
if (size < 0) ERR("ftell");
rewind(f);
```

Zapis za końcem pliku (po `fseek` dalej niż rozmiar) tworzy dziurę wypełnioną zerami.

### 7.6 `feof`, `ferror`, `fflush`

- `int feof(FILE *f)`: niezerowe, gdy próbowano czytać za końcem. Nie używaj `while (!feof(f))` jako warunku pętli; sprawdzaj wynik funkcji czytającej.
- `int ferror(FILE *f)`: niezerowe, gdy wystąpił błąd odczytu/zapisu.
- `int fflush(FILE *f)`: wypycha bufor do pliku teraz. `0` sukces, `EOF` błąd.

### 7.7 Buforowanie stdout i stderr

| Strumień | Do terminala | Do pliku / potoku |
| --- | --- | --- |
| `stdout` | buforowany liniowo: wypisuje przy `\n` | buforowany całkowicie: przy zapełnieniu bufora (\~4 KB) lub końcu programu |
| `stderr` | niebuforowany: od razu | niebuforowany: od razu |

- `printf("%d", i)` bez `\n` w pętli ze `sleep` nic nie pokaże, dopóki nie pojawi się `\n` albo `fflush(stdout)`.
- `./prog > out.txt` + Ctrl+C może zostawić pusty plik, bo bufor nie został wypchnięty.
- Komunikaty diagnostyczne i błędy pisz na `stderr`, a wyniki na `stdout`. Przekierowanie stderr w terminalu to `2>` (np. `./prog 2> err.txt`).

## 8. Katalogi: `opendir`, `readdir`, `closedir` `<dirent.h>`

Katalog czytasz jak plik: otwierasz `opendir`, w pętli bierzesz kolejne wpisy `readdir` aż do `NULL`, zamykasz `closedir`. Wpis daje tylko nazwę i numer i-węzła; resztę (typ, rozmiar) bierzesz z `lstat`.

### 8.1 Funkcje

| Funkcja | Sygnatura | Sukces | Błąd |
| --- | --- | --- | --- |
| `opendir` | `DIR *opendir(const char *dirname)` | wskaźnik `DIR*` | `NULL` + `errno` |
| `readdir` | `struct dirent *readdir(DIR *dirp)` | kolejny wpis; `NULL` = koniec | `NULL` + `errno ≠ 0` |
| `closedir` | `int closedir(DIR *dirp)` | `0` | `-1` |
| `rewinddir` | `void rewinddir(DIR *dirp)` | wraca na początek katalogu | brak |
| `fdopendir` | `DIR *fdopendir(int fd)` | `DIR*` z deskryptora katalogu | `NULL` |

**`struct dirent`** (POSIX gwarantuje tylko te dwa pola):

```c
ino_t d_ino;      // numer i-węzła
char  d_name[];   // nazwa wpisu (BEZ ścieżki katalogu!), zakończona '\0'
```

Linux dodaje `d_off`, `d_reclen` i `d_type` (`DT_REG`, `DT_DIR`, `DT_LNK`...). Na labach trzymaj się standardu i typ bierz z `lstat`.

### 8.2 Wzorzec pętli (zapamiętaj)

```c
DIR *dir;
struct dirent *entry;

if (!(dir = opendir(path)))
    ERR("opendir");

errno = 0;                                  // readdir nie zeruje errno sam
while ((entry = readdir(dir)) != NULL)
{
    if (!strcmp(entry->d_name, ".") || !strcmp(entry->d_name, ".."))
        continue;                           // pomiń "ten" i "nadrzędny" katalog

    printf("%s\n", entry->d_name);

    errno = 0;                              // zeruj przed KAŻDYM kolejnym readdir
}
if (errno)                                  // NULL był błędem, nie końcem
    ERR("readdir");

if (closedir(dir))
    ERR("closedir");
```

Wersja z tutorialu (`do { errno = 0; if ((dp = readdir(dirp)) != NULL) {...} } while (dp != NULL);`) robi to samo.

### 8.3 Pułapki

- `d_name` to sama nazwa. Jeśli katalog nie jest bieżącym (`.`), do `lstat` musisz podać `"katalog/nazwa"` (rozdział 10) albo najpierw zrobić `chdir(katalog)`.
- Każdy katalog zawiera `.` i `..`. Pomiń je, bo inaczej liczniki się nie zgodzą, a rekurencja się zapętli.
- Kolejność wpisów jest dowolna (nie alfabetyczna).
- Wskaźnik z `readdir` jest ważny tylko do następnego `readdir`/`closedir`. Gdy nazwa ma zostać na później, zrób `strdup(entry->d_name)`.
- Nie zwalniaj wyniku `readdir`. Zwalniasz tylko `DIR*` przez `closedir`.
- Każdy otwarty `DIR*` zajmuje deskryptor. W rekurencji zamykaj katalogi, a limit otwartych katalogów pilnuje `nftw`.

### 8.4 Pliki przed podkatalogami (dwa przejścia)

```c
// przejście 1: pliki, przejście 2: rekurencja w katalogi
for (int pass = 1; pass <= 2; pass++)
{
    if (pass == 2)
        rewinddir(dir);
    errno = 0;
    while ((entry = readdir(dir)) != NULL) { /* ... */ errno = 0; }
    if (errno)
        ERR("readdir");
}
```

## 9. Informacje o pliku: `stat`, `lstat`, `fstat` `<sys/stat.h>`

`stat` wypełnia strukturę `struct stat` danymi o pliku: typ, uprawnienia, rozmiar, właściciel, daty. `lstat` robi to samo, ale dla dowiązania symbolicznego opisuje SAM link, a nie plik docelowy.

### 9.1 Funkcje

```c
int stat(const char *restrict path, struct stat *restrict buf);   // idzie za symlinkiem
int lstat(const char *restrict path, struct stat *restrict buf);  // NIE idzie za symlinkiem
int fstat(int fd, struct stat *buf);                              // dla otwartego deskryptora
// 0 = sukces, -1 = błąd (ENOENT, EACCES, ...)
```

`buf` to adres TWOJEJ zmiennej: `struct stat st; stat(path, &st);`. Nie przekazuj niezainicjowanego wskaźnika `struct stat *st;`.

### 9.2 Najważniejsze pola `struct stat`

| Pole | Typ | Znaczenie |
| --- | --- | --- |
| `st_mode` | `mode_t` | typ pliku + bity uprawnień |
| `st_size` | `off_t` | rozmiar w bajtach (dla linku: długość ścieżki docelowej) |
| `st_ino` | `ino_t` | numer i-węzła |
| `st_nlink` | `nlink_t` | liczba dowiązań twardych |
| `st_uid` / `st_gid` | `uid_t` / `gid_t` | właściciel / grupa |
| `st_dev` | `dev_t` | urządzenie, na którym leży plik |
| `st_blocks` | `blkcnt_t` | zajęte bloki 512 B (pliki z dziurami mają mniej niż rozmiar) |
| `st_mtime` | `time_t` | ostatnia modyfikacja |
| `st_atime` / `st_ctime` | `time_t` | ostatni dostęp / zmiana metadanych |

### 9.3 Makra typu pliku (argument: `st.st_mode`)

| Makro | Prawda gdy |
| --- | --- |
| `S_ISREG(m)` | zwykły plik |
| `S_ISDIR(m)` | katalog |
| `S_ISLNK(m)` | dowiązanie symboliczne (tylko z `lstat`!) |
| `S_ISFIFO(m)` | potok nazwany |
| `S_ISCHR(m)` / `S_ISBLK(m)` | urządzenie znakowe / blokowe |
| `S_ISSOCK(m)` | gniazdo |

```c
struct stat st;
if (lstat(path, &st))
    ERR("lstat");
if (S_ISDIR(st.st_mode))       dirs++;
else if (S_ISREG(st.st_mode))  files++;
else if (S_ISLNK(st.st_mode))  links++;
else                           other++;

printf("%s %lld\n", path, (long long)st.st_size);
```

Z `stat` makro `S_ISLNK` nigdy nie będzie prawdziwe, bo `stat` przechodzi przez link. Do liczenia linków zawsze `lstat`.

### 9.4 Bity uprawnień

Uprawnienia to 9 bitów zapisywanych ósemkowo: trzy cyfry dla właściciela, grupy i reszty, każda to suma r=4, w=2, x=1.

| Ósemkowo | Symbol | Znaczenie |
| --- | --- | --- |
| `0400` / `0200` / `0100` | `S_IRUSR` / `S_IWUSR` / `S_IXUSR` | właściciel: odczyt / zapis / wykonanie |
| `0040` / `0020` / `0010` | `S_IRGRP` / `S_IWGRP` / `S_IXGRP` | grupa |
| `0004` / `0002` / `0001` | `S_IROTH` / `S_IWOTH` / `S_IXOTH` | pozostali |
| `0700` / `0070` / `0007` | `S_IRWXU` / `S_IRWXG` / `S_IRWXO` | wszystkie trzy dla danej klasy |

Przykłady: `0644` = `rw-r--r--` (zwykły plik), `0755` = `rwxr-xr-x` (katalog, program), `0444` = `r--r--r--` (tylko odczyt dla wszystkich), `0600` = `rw-------`. Początkowe `0` w C oznacza zapis ósemkowy: `0644` ≠ `644`.

```c
mode_t perms = st.st_mode & 0777;           // same uprawnienia, bez typu
printf("%o\n", perms);                      // np. 644
if (st.st_mode & S_IWUSR) ...               // czy właściciel może pisać
```

## 10. Katalog roboczy i ścieżki

Nazwa z `readdir` to tylko nazwa. Do `lstat`/`fopen` potrzebujesz ścieżki, którą albo sklejasz (`katalog/nazwa`), albo używasz bezpośrednio po `chdir(katalog)`.

### 10.1 `getcwd` i `chdir` `<unistd.h>`

```c
char *getcwd(char *buf, size_t size);   // wpisuje BEZWZGLĘDNĄ ścieżkę katalogu roboczego; zwraca buf albo NULL
int chdir(const char *path);            // zmienia katalog roboczy procesu; 0 albo -1
```

- `buf` musisz zaalokować sam (np. `char cwd[PATH_MAX];`). Za mały bufor da `NULL` z `ERANGE`.
- `chdir` działa na cały proces: po nim wszystkie ścieżki względne są liczone od nowego katalogu.
- Zmiana katalogu nie wpływa na powłokę, z której uruchomiłeś program.

```c
// wzorzec z tutorialu: wejdź, przetwórz, WRÓĆ
char cwd[PATH_MAX];
if (!getcwd(cwd, sizeof(cwd)))
    ERR("getcwd");
for (int i = 1; i < argc; i++)
{
    if (chdir(argv[i])) {                  // nieistniejący katalog: komunikat i dalej
        fprintf(stderr, "%s: %s\n", argv[i], strerror(errno));
        continue;
    }
    scan_dir();                            // pracuje na "."
    if (chdir(cwd))                        // powrót, żeby następna ścieżka względna była poprawna
        ERR("chdir");
}
```

### 10.2 Sklejanie ścieżek

`PATH_MAX` z `<limits.h>` to maksymalna długość ścieżki (w Linuksie 4096).

```c
// A) bufor na stosie + snprintf (najprościej)
char full[PATH_MAX];
if (snprintf(full, sizeof(full), "%s/%s", dir, entry->d_name) >= (int)sizeof(full))
    ERR("path too long");

// B) nowa pamięć (wynik do free), obsługuje dir z '/' na końcu
char *join_paths(const char *a, const char *b)
{
    size_t la = strlen(a);
    int slash = (la > 0 && a[la - 1] == '/') ? 0 : 1;
    char *res = malloc(la + slash + strlen(b) + 1);
    if (!res)
        ERR("malloc");
    strcpy(res, a);
    if (slash)
        strcat(res, "/");
    return strcat(res, b);
}

char *p = join_paths("index/by-title", title);
/* ... */
free(p);
```

### 10.3 Ścieżki względne i bezwzględne

| Ścieżka | Liczona od | Przykład |
| --- | --- | --- |
| bezwzględna | korzenia `/` | `/var/log` |
| względna | katalogu roboczego | `dir1`, `./dir1`, `../a/file.txt` |
| `.` | ten katalog | `opendir(".")` |
| `..` | katalog nadrzędny |  |
| `~` | rozwija TYLKO powłoka, nie C | `fopen("~/x")` nie zadziała |

Nazwa pliku ze ścieżki: `strrchr(path, '/')` (rozdział 3.5). `nftw` podaje ją od razu jako `fpath + ftwbuf->base` (rozdział 13).

## 11. Tworzenie, usuwanie, uprawnienia

Wszystkie funkcje z tego rozdziału zwracają `0` przy sukcesie i `-1` z ustawionym `errno` przy błędzie, oprócz `umask`.

### 11.1 Szybka ściąga

| Funkcja | Nagłówek | Sygnatura | Co robi | Typowe `errno` |
| --- | --- | --- | --- | --- |
| `mkdir` | `<sys/stat.h>` | `int mkdir(const char *path, mode_t mode)` | tworzy pusty katalog | `EEXIST`, `ENOENT` (brak rodzica) |
| `rmdir` | `<unistd.h>` | `int rmdir(const char *path)` | usuwa PUSTY katalog | `ENOTEMPTY` |
| `unlink` | `<unistd.h>` | `int unlink(const char *path)` | usuwa plik (nazwę) | `ENOENT`, `EISDIR`/`EPERM` dla katalogu |
| `remove` | `<stdio.h>` | `int remove(const char *path)` | `unlink` dla pliku, `rmdir` dla katalogu |  |
| `rename` | `<stdio.h>` | `int rename(const char *old, const char *new)` | zmienia nazwę / przenosi; nadpisuje `new` | `ENOENT`, `EXDEV` (inny dysk) |
| `chmod` | `<sys/stat.h>` | `int chmod(const char *path, mode_t mode)` | ustawia uprawnienia | `EPERM` (nie Twój plik) |
| `access` | `<unistd.h>` | `int access(const char *path, int amode)` | sprawdza istnienie / prawa | `ENOENT`, `EACCES` |
| `umask` | `<sys/stat.h>` | `mode_t umask(mode_t mask)` | ustawia maskę tworzenia, zwraca STARĄ | nie zawodzi |

### 11.2 `mkdir`

```c
if (mkdir("index", 0755))                    // błąd, gdy już istnieje
    ERR("mkdir");

if (mkdir(genre_dir, 0755) && errno != EEXIST) // "utwórz, jeśli nie ma"
    ERR("mkdir");
```

Katalog potrzebuje prawa `x`, żeby dało się do niego wejść i dostać do plików. Używaj `0755` lub `0777`. `mkdir` nie tworzy brakujących katalogów nadrzędnych: `a/b/c` wymaga trzech wywołań.

### 11.3 `unlink`, `rename`

```c
if (unlink(name) && errno != ENOENT)   // usuń, jeśli istnieje
    ERR("unlink");

// bezpieczna podmiana pliku: zapis do tmp, potem rename
if (rename("env/requirements.tmp", "env/requirements"))
    ERR("rename");
```

- Usunięcie pliku zależy od praw do KATALOGU (`w` + `x`), nie do pliku. Plik `r--r--r--` można usunąć.
- Gdy ktoś ma plik otwarty, `unlink` usuwa tylko nazwę. Dane znikają po zamknięciu ostatniego deskryptora.

### 11.4 `chmod` i `access`

```c
if (chmod(path, 0444))                 // tylko odczyt dla wszystkich
    ERR("chmod");

if (access(path, F_OK) == 0)           // czy istnieje
    ...
if (access(path, R_OK | W_OK))         // czy mogę czytać I pisać; -1 = nie
    fprintf(stderr, "no access to %s\n", path);
```

`amode` w `access` to flagi `F_OK`, `R_OK`, `W_OK`, `X_OK`, a nie bity ósemkowe jak w `chmod`.

### 11.5 `umask`

Maska mówi, które bity uprawnień ZABRAĆ przy tworzeniu pliku: faktyczne prawa = `mode & ~umask`. Domyślna maska to zwykle `022`, więc `fopen` (0666) daje `0644`, a `mkdir(..., 0777)` daje `0755`.

```c
// chcę, żeby fopen stworzył plik z prawami perms (np. 0640)
umask(~perms & 0777);       // zabierz wszystko, czego NIE ma w perms
FILE *f = fopen(name, "w+"); // 0666 & ~(~0640 & 0777) = 0640
```

- `~perms & 0777` to dopełnienie `perms` do 9 bitów: bity, których nie chcemy.
- `fopen` startuje z `0666`, więc przez `umask` nie dostaniesz `x`. Do praw z `x` użyj `open(name, O_CREAT|..., 0755)` albo `chmod` po utworzeniu.
- `umask` zmienia maskę tylko w Twoim procesie. Nie sprawdzasz jej błędów, bo ich nie ma.

## 12. Dowiązania: `symlink`, `readlink`, `link` `<unistd.h>`

Dowiązanie symboliczne to mały plik zawierający ścieżkę do innego pliku. Względna ścieżka w symlinku jest liczona od katalogu, w którym leży SAM LINK, a nie od katalogu roboczego programu.

### 12.1 Funkcje

| Funkcja | Sygnatura | Co robi | Zwraca |
| --- | --- | --- | --- |
| `symlink` | `int symlink(const char *target, const char *linkpath)` | tworzy `linkpath -> target` | `0` / `-1` (`EEXIST`, gdy nazwa zajęta) |
| `readlink` | `ssize_t readlink(const char *restrict path, char *restrict buf, size_t bufsize)` | czyta ścieżkę zapisaną w linku | liczbę bajtów albo `-1`; NIE dopisuje `'\0'` |
| `link` | `int link(const char *existing, const char *new)` | tworzy dowiązanie twarde (drugą nazwę tego samego i-węzła) | `0` / `-1` |

Kolejność argumentów `symlink` jest taka sama jak w `ln -s CEL LINK`: najpierw na co wskazuje, potem gdzie powstaje link.

### 12.2 Ścieżka względna w symlinku

```text
.
├── a/file.txt
└── b/

ln -s a/file.txt    b/file.txt_bak   # ŹLE: z b/ ścieżka a/file.txt nie istnieje (b/a/file.txt)
ln -s ../a/file.txt b/file.txt_bak   # DOBRZE: z b/ wychodzimy do .. i wchodzimy w a/
```

Reguła: dodaj tyle `../`, ile poziomów katalogów ma ścieżka do linku.

```c
// link w index/by-title/<tytuł> (2 poziomy) -> plik library/a/b/zwój
char target[PATH_MAX], linkpath[PATH_MAX];
snprintf(target, sizeof(target), "../../%s", fpath);        // "../../library/a/b/zwój"
snprintf(linkpath, sizeof(linkpath), "index/by-title/%s", title);
if (symlink(target, linkpath))
{
    if (errno == EEXIST)
        fprintf(stderr, "duplicate: %s\n", title);
    ERR("symlink");
}

// link w index/by-genre/<gatunek>/<tytuł> (3 poziomy) -> "../../../library/..."
```

Alternatywa: ścieżka bezwzględna (`getcwd` + `/` + `fpath`). Działa niezależnie od położenia linku, ale psuje się po przeniesieniu całego katalogu.

### 12.3 `readlink`

```c
char buf[PATH_MAX];
ssize_t n = readlink(path, buf, sizeof(buf) - 1);   // zostaw miejsce na '\0'
if (n < 0)
    ERR("readlink");
buf[n] = '\0';                                      // MUSISZ dopisać sam
printf("%s -> %s\n", path, buf);
```

### 12.4 Twarde a symboliczne

|  | Twarde (`link`) | Symboliczne (`symlink`) |
| --- | --- | --- |
| Czym jest | kolejną nazwą tego samego i-węzła | osobnym plikiem ze ścieżką |
| Po usunięciu oryginału | dalej działa | wisi (wskazuje w pustkę) |
| Na katalog / inny dysk | nie | tak |
| `lstat` widzi | zwykły plik, `st_nlink` > 1 | `S_ISLNK` |
| Na labach | raczej nie używamy | używamy |

## 13. Rekurencyjne przeglądanie: `nftw` `<ftw.h>`

`nftw` przechodzi całe drzewo katalogów i dla każdego obiektu wywołuje Twoją funkcję. Sama otwiera i zamyka katalogi, więc nie piszesz rekurencji ani pętli `readdir`.

### 13.1 Sygnatura

```c
int nftw(const char *path,
         int (*fn)(const char *fpath, const struct stat *sb, int typeflag, struct FTW *ftwbuf),
         int fd_limit,
         int flags);
```

- `path`: katalog startowy.
- `fn`: Twoja funkcja (callback), wywoływana dla każdego obiektu, łącznie z samym `path`.
- `fd_limit`: ile katalogów może być otwartych naraz (np. `20`). Nie ogranicza głębokości.
- `flags`: suma bitowa flag poniżej (albo `0`).
- Zwraca `0`, gdy przeszła wszystko; `-1` przy błędzie; albo pierwszą niezerową wartość zwróconą przez `fn` (to przerywa przechodzenie).

### 13.2 Argumenty callbacku

| Argument | Znaczenie |
| --- | --- |
| `fpath` | pełna ścieżka obiektu, np. `library/a/b/zwój` (zaczyna się od `path`) |
| `sb` | gotowy wynik `stat`/`lstat` dla obiektu (rozmiar: `sb->st_size`) |
| `typeflag` | typ obiektu, tabela niżej |
| `ftwbuf->base` | indeks, od którego w `fpath` zaczyna się sama nazwa: `fpath + ftwbuf->base` |
| `ftwbuf->level` | głębokość: `0` = sam `path`, `1` = jego zawartość, ... |

| `typeflag` | Znaczenie |
| --- | --- |
| `FTW_F` | zwykły plik (i inne nie-katalogi) |
| `FTW_D` | katalog (przed jego zawartością) |
| `FTW_DP` | katalog po zawartości (tylko z `FTW_DEPTH`) |
| `FTW_SL` | dowiązanie symboliczne (z `FTW_PHYS`) |
| `FTW_SLN` | link wiszący (bez `FTW_PHYS`) |
| `FTW_DNR` | katalogu nie da się czytać (brak praw) |
| `FTW_NS` | `stat` się nie udał, `sb` nieważne |

| Flaga | Efekt |
| --- | --- |
| `FTW_PHYS` | NIE wchodzi w symlinki (używa `lstat`); POLECANA, chroni przed pętlami |
| `FTW_DEPTH` | najpierw zawartość, potem katalog (do kasowania drzewa) |
| `FTW_MOUNT` | nie przechodzi na inne systemy plików |
| `FTW_CHDIR` | robi `chdir` do każdego katalogu przed callbackiem |

### 13.3 Wzorzec

```c
// callback: zwróć 0 = idź dalej
int walk(const char *fpath, const struct stat *sb, int type, struct FTW *f)
{
    if (type == FTW_D)
        printf("dir  %s (poziom %d)\n", fpath, f->level);
    else if (type == FTW_F)
        printf("file %s %lld, nazwa: %s\n", fpath, (long long)sb->st_size, fpath + f->base);
    return 0;
}

int main(void)
{
    if (nftw("library", walk, 20, FTW_PHYS))
        ERR("nftw");
    return EXIT_SUCCESS;
}
```

- Callback nie dostaje własnych argumentów. Dane, które musi znać (np. maks. głębokość, licznik), trzymasz w zmiennej globalnej – to jedyny dopuszczalny wyjątek.
- Ograniczenie głębokości: `nftw` i tak wejdzie głębiej, ale możesz pomijać obiekty z `f->level > max_depth`. Gdy masz też wypisywać „katalog, a potem jego pliki”, łatwiej o własną rekurencję z `opendir` (przepis 16.6).
- Wzorzec licznika z globalną zmienną:

```c
static int files = 0, dirs = 0;
int count(const char *p, const struct stat *s, int t, struct FTW *f)
{
    (void)p; (void)s; (void)f;          // nieużywane argumenty: wycisza -Wextra
    if (t == FTW_D) dirs++;
    else if (t == FTW_F) files++;
    return 0;
}
```

`ftw` (bez „n”) to starsza, prostsza wersja bez `flags` i bez `struct FTW`. Na labach używaj `nftw`.

## 14. Niskopoziomowe I/O: `open`, `read`, `write`, `close`, `lseek`, `readv`

Deskryptor to liczba `int` (0 = stdin, 1 = stdout, 2 = stderr, kolejne z `open`). Funkcje działają bez bufora biblioteki, są bezpośrednimi wywołaniami systemu i dają pełną kontrolę nad uprawnieniami (`open` z `O_CREAT`).

### 14.1 `open` i `close` `<fcntl.h>`, `<unistd.h>`

```c
int open(const char *path, int oflag, ... /* mode_t mode */);   // deskryptor >= 0 albo -1
int close(int fd);                                             // 0 albo -1
```

| Flaga | Znaczenie |
| --- | --- |
| `O_RDONLY` / `O_WRONLY` / `O_RDWR` | tylko odczyt / tylko zapis / oba (dokładnie jedna) |
| `O_CREAT` | utwórz, jeśli nie istnieje; WYMAGA trzeciego argumentu `mode` |
| `O_EXCL` | z `O_CREAT`: błąd `EEXIST`, jeśli plik już jest |
| `O_TRUNC` | obetnij istniejący plik do 0 |
| `O_APPEND` | każdy zapis na końcu pliku |

```c
int fd = open(path, O_WRONLY | O_CREAT | O_EXCL, 0444);  // nowy plik r--r--r--, błąd gdy istnieje
if (fd < 0)
{
    if (errno == EEXIST)
        fprintf(stderr, "%s already exists\n", path);
    ERR("open");
}
/* write(fd, ...) działa, mimo że prawa to 0444: plik właśnie stworzyłeś do zapisu */
if (close(fd))
    ERR("close");
```

Odpowiedniki `fopen`: `"r"` = `O_RDONLY`, `"w"` = `O_WRONLY|O_CREAT|O_TRUNC, 0666`, `"a"` = `O_WRONLY|O_CREAT|O_APPEND, 0666`. `mode` w `open` też jest pomniejszany o `umask`.

### 14.2 `read`, `write`, `lseek`

```c
ssize_t read(int fd, void *buf, size_t n);         // przeczytane bajty; 0 = koniec pliku; -1 = błąd
ssize_t write(int fd, const void *buf, size_t n);  // zapisane bajty (może być < n!); -1 = błąd
off_t lseek(int fd, off_t offset, int whence);     // nowa pozycja albo -1; whence jak w fseek
```

`read` i `write` mogą przetworzyć MNIEJ bajtów niż chcesz. Pewny sposób to pętla:

```c
ssize_t bulk_read(int fd, char *buf, size_t count)
{
    size_t done = 0;
    while (done < count)
    {
        ssize_t c = read(fd, buf + done, count - done);
        if (c < 0)
            return -1;          // błąd
        if (c == 0)
            break;              // koniec pliku
        done += c;
    }
    return done;
}

ssize_t bulk_write(int fd, const char *buf, size_t count)
{
    size_t done = 0;
    while (done < count)
    {
        ssize_t c = write(fd, buf + done, count - done);
        if (c < 0)
            return -1;
        done += c;
    }
    return done;
}
```

### 14.3 `readv` i `writev` `<sys/uio.h>`

Jedno wywołanie czyta (pisze) do kilku buforów po kolei. Przydaje się do rekordów binarnych z polami różnych typów.

```c
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

struct iovec {
    void  *iov_base;   // adres bufora
    size_t iov_len;    // ile bajtów do/z niego
};
// zwraca łączną liczbę bajtów, 0 = koniec, -1 = błąd
```

```c
// rekord: 4 bajty unsigned int + 64 bajty tytułu (dopełnione zerami)
typedef struct {
    unsigned int size;
    char title[65];          // 65: ostatni bajt zawsze '\0'
} entry_t;

entry_t e;
e.title[64] = '\0';
struct iovec iov[2] = {
    { .iov_base = &e.size, .iov_len = sizeof(e.size) },  // 4
    { .iov_base = e.title, .iov_len = 64 },
};

int fd = open(db_path, O_RDONLY);
if (fd < 0)
    ERR("open");

struct stat st;
if (fstat(fd, &st))
    ERR("fstat");
if (st.st_size % 68) {                    // liczba rekordów = rozmiar / 68
    fprintf(stderr, "bad database size\n");  // nie ERR: errno nie jest tu ustawione
    exit(EXIT_FAILURE);
}

for (off_t i = 0; i < st.st_size / 68; i++)
{
    ssize_t n = readv(fd, iov, 2);
    if (n < 0)
        ERR("readv");
    if (n != 68)
        ERR("short read");
    printf("%s %u\n", e.title, e.size);
}
if (close(fd))
    ERR("close");
```

## 15. Środowisko, losowość, czas

### 15.1 Zmienne środowiskowe `<stdlib.h>`

```c
char *getenv(const char *name);                          // wartość albo NULL, gdy brak; NIE zwalniaj
int setenv(const char *name, const char *value, int overwrite);   // 0 albo -1
int unsetenv(const char *name);                          // 0 albo -1
```

```c
FILE *out = stdout;
char *fname = getenv("L1_OUTPUTFILE");
if (to_file && fname)                    // -o i zmienna istnieje
{
    if (!(out = fopen(fname, "w")))      // "w" nadpisze istniejący plik
        ERR("fopen");
}
```

W terminalu: `export L1_OUTPUTFILE=out.txt` albo jednorazowo `L1_OUTPUTFILE=out.txt ./prog -p . -o`. Podgląd: `echo $L1_OUTPUTFILE`, `env`.

### 15.2 Liczby losowe

```c
void srand(unsigned seed);   // ustawia ziarno: wywołaj RAZ na początku programu
int rand(void);              // liczba z zakresu 0..RAND_MAX

srand(time(NULL));                         // ziarno = bieżący czas w sekundach
int x = rand() % 100;                      // 0..99
int y = 10 + rand() % 91;                  // 10..100
char c = 'A' + rand() % ('Z' - 'A' + 1);   // losowa wielka litera
long pos = rand() % size;                  // losowa pozycja w pliku
```

Dwa uruchomienia w tej samej sekundzie dadzą te same liczby. Lepsze ziarno to `srand(time(NULL) ^ getpid());`.

### 15.3 Czas i czekanie

```c
#include <time.h>
time_t time(time_t *t);              // sekundy od 1970; zwykle time(NULL)
char *ctime(const time_t *t);        // napis z datą, np. dla st.st_mtime

#include <unistd.h>
unsigned sleep(unsigned seconds);    // uśpij proces

printf("modyfikacja: %s", ctime(&st.st_mtime));   // ctime ma już '\n'
```

## 16. Gotowe przepisy

Każdy przepis to sprawdzona funkcja z typowego zadania L1. Wszystkie zakładają szablon z rozdziału 1 (nagłówki + `ERR`).

### 16.1 Zliczanie obiektów w katalogu

```c
void count_dir(const char *path)
{
    DIR *dir;
    struct dirent *e;
    struct stat st;
    char full[PATH_MAX];
    int files = 0, dirs = 0, links = 0, other = 0;

    if (!(dir = opendir(path)))
        ERR("opendir");
    errno = 0;
    while ((e = readdir(dir)) != NULL)
    {
        if (!strcmp(e->d_name, ".") || !strcmp(e->d_name, ".."))
            continue;
        snprintf(full, sizeof(full), "%s/%s", path, e->d_name);
        if (lstat(full, &st))
            ERR("lstat");
        if (S_ISREG(st.st_mode)) files++;
        else if (S_ISDIR(st.st_mode)) dirs++;
        else if (S_ISLNK(st.st_mode)) links++;
        else other++;
        errno = 0;
    }
    if (errno)
        ERR("readdir");
    if (closedir(dir))
        ERR("closedir");
    printf("Files: %d, Dirs: %d, Links: %d, Other: %d\n", files, dirs, links, other);
}
```

### 16.2 Listing z rozmiarami, błąd jednego katalogu nie przerywa reszty

```c
// zwraca 0 albo -1, gdy katalogu nie da się otworzyć
int list_dir(const char *path, FILE *out)
{
    DIR *dir = opendir(path);
    if (!dir) {
        fprintf(stderr, "cannot open directory %s: %s\n", path, strerror(errno));
        return -1;
    }
    fprintf(out, "SCIEZKA:\n%s\nLISTA PLIKÓW:\n", path);
    struct dirent *e;
    struct stat st;
    char full[PATH_MAX];
    errno = 0;
    while ((e = readdir(dir)) != NULL) {
        if (strcmp(e->d_name, ".") && strcmp(e->d_name, "..")) {
            snprintf(full, sizeof(full), "%s/%s", path, e->d_name);
            if (lstat(full, &st))
                ERR("lstat");
            fprintf(out, "%s %lld\n", e->d_name, (long long)st.st_size);
        }
        errno = 0;
    }
    if (errno)
        ERR("readdir");
    if (closedir(dir))
        ERR("closedir");
    return 0;
}

// w main: FILE *out = out_name ? fopen(out_name, "w") : stdout;  (+ sprawdzenie NULL)
```

### 16.3 Parser pliku `klucz:wartość`

```c
typedef struct { char *author, *title, *genre; } book_t;   // brak pola = NULL

static void set_field(char **field, const char *value)
{
    char *copy = strdup(value);
    if (!copy)
        ERR("strdup");
    free(*field);                 // gdyby klucz wystąpił drugi raz
    *field = copy;
}

int read_book(const char *path, book_t *b)
{
    b->author = b->title = b->genre = NULL;
    FILE *f = fopen(path, "r");
    if (!f)
        return -1;
    char *line = NULL;
    size_t cap = 0;
    ssize_t n;
    while ((n = getline(&line, &cap, f)) != -1)
    {
        if (n > 0 && line[n - 1] == '\n') line[--n] = '\0';
        if (n > 0 && line[n - 1] == '\r') line[--n] = '\0';
        char *sep = strchr(line, ':');
        if (!sep)                 // linia bez ':' (np. kontynuacja) -> pomiń
            continue;
        *sep = '\0';
        if (!strcmp(line, "author")) set_field(&b->author, sep + 1);
        else if (!strcmp(line, "title")) set_field(&b->title, sep + 1);
        else if (!strcmp(line, "genre")) set_field(&b->genre, sep + 1);
    }
    if (ferror(f))
        ERR("getline");
    free(line);
    if (fclose(f))
        ERR("fclose");
    return 0;
}

void free_book(book_t *b) { free(b->author); free(b->title); free(b->genre); }

// wypisanie: printf("author: %s\n", b.author ? b.author : "missing!");
```

### 16.4 Wczytanie całego pliku do pamięci

```c
char *read_all(const char *path, long *out_size)
{
    FILE *f = fopen(path, "r");
    if (!f)
        ERR("fopen");
    if (fseek(f, 0, SEEK_END))
        ERR("fseek");
    long size = ftell(f);
    if (size < 0)
        ERR("ftell");
    rewind(f);
    char *buf = malloc(size + 1);         // +1 na '\0', żeby dało się użyć jak napisu
    if (!buf)
        ERR("malloc");
    if (fread(buf, 1, size, f) != (size_t)size)
        ERR("fread");
    buf[size] = '\0';
    if (fclose(f))
        ERR("fclose");
    *out_size = size;
    return buf;                           // wywołujący robi free
}
```

### 16.5 Usunięcie linii z pliku (przez plik tymczasowy)

```c
void remove_line(const char *path, const char *prefix)
{
    char tmp[PATH_MAX];
    snprintf(tmp, sizeof(tmp), "%s.tmp", path);
    FILE *in = fopen(path, "r"), *out;
    if (!in)
        ERR("fopen");
    if (!(out = fopen(tmp, "w")))
        ERR("fopen");
    char *line = NULL;
    size_t cap = 0;
    size_t plen = strlen(prefix);
    while (getline(&line, &cap, in) != -1)
        if (!(strncmp(line, prefix, plen) == 0 && line[plen] == ' '))  // przepisz wszystkie OPRÓCZ pasującej
            if (fputs(line, out) == EOF)
                ERR("fputs");
    if (ferror(in))
        ERR("getline");
    free(line);
    if (fclose(in) || fclose(out))
        ERR("fclose");
    if (rename(tmp, path))                // podmiana
        ERR("rename");
}
```

### 16.6 Własna rekurencja z ograniczeniem głębokości i filtrem rozszerzenia

```c
int has_ext(const char *name, const char *ext)
{
    if (!ext) return 1;                   // brak filtra
    const char *dot = strrchr(name, '.');
    return dot && !strcmp(dot + 1, ext);
}

// depth == 1: tylko ten katalog
void scan(const char *path, int depth, const char *ext, FILE *out)
{
    DIR *dir = opendir(path);
    if (!dir)
        ERR("opendir");
    fprintf(out, "path: %s\n", path);
    struct dirent *e;
    struct stat st;
    char full[PATH_MAX];
    for (int pass = 1; pass <= 2; pass++)  // 1: pliki, 2: podkatalogi
    {
        if (pass == 2) {
            if (depth <= 1) break;
            rewinddir(dir);
        }
        errno = 0;
        while ((e = readdir(dir)) != NULL)
        {
            if (strcmp(e->d_name, ".") && strcmp(e->d_name, ".."))
            {
                snprintf(full, sizeof(full), "%s/%s", path, e->d_name);
                if (lstat(full, &st))
                    ERR("lstat");
                if (pass == 1 && S_ISREG(st.st_mode) && has_ext(e->d_name, ext))
                    fprintf(out, "%s %lld\n", e->d_name, (long long)st.st_size);
                else if (pass == 2 && S_ISDIR(st.st_mode))
                    scan(full, depth - 1, ext, out);
            }
            errno = 0;
        }
        if (errno)
            ERR("readdir");
    }
    if (closedir(dir))
        ERR("closedir");
}
```

### 16.7 Plik o zadanych prawach i DOKŁADNYM rozmiarze

```c
void make_file(const char *name, long size, mode_t perms, int percent)
{
    if (unlink(name) && errno != ENOENT)   // stary plik zachowałby stare prawa
        ERR("unlink");
    umask(~perms & 0777);
    FILE *f = fopen(name, "w+");
    if (!f)
        ERR("fopen");
    for (long i = 0; i < size * percent / 100; i++)
    {
        if (fseek(f, rand() % size, SEEK_SET))
            ERR("fseek");
        fputc('A' + rand() % 26, f);
    }
    if (size > 0) {                        // dopisz zero na ostatniej pozycji: rozmiar zawsze == size
        if (fseek(f, size - 1, SEEK_SET))
            ERR("fseek");
        fputc('\0', f);
    }
    if (fclose(f))
        ERR("fclose");
}
```

### 16.8 Symlink do każdego pliku z drzewa (`nftw`)

```c
int add_link(const char *fpath, const struct stat *sb, int type, struct FTW *f)
{
    (void)sb;
    if (type != FTW_F)
        return 0;
    char target[PATH_MAX], linkp[PATH_MAX];
    snprintf(target, sizeof(target), "../../%s", fpath);               // link leży 2 poziomy niżej
    snprintf(linkp, sizeof(linkp), "index/by-visible-title/%s", fpath + f->base);
    if (symlink(target, linkp))
        ERR("symlink");                    // EEXIST = powtórzona nazwa
    return 0;
}

// main: mkdir("index", 0755); mkdir("index/by-visible-title", 0755);
//       if (nftw("library", add_link, 20, FTW_PHYS)) ERR("nftw");
```

## 17. Odpowiedzi na pytania z tutorialu L1

Odpowiedzi do pytań, które w tutorialu zostały puste, oraz krótkie przypomnienie tych z gotową odpowiedzią.

### Przeglądanie katalogu (l1-1)

1. **Katalog bez podkatalogów pokazuje 2 katalogi. Dlaczego?** Program liczy `.` i `..`. Każdy katalog ma te dwa wpisy, więc pomijaj je przez `strcmp`.
2. **Jak utworzyć link symboliczny do testów?** `ln -s plik.txt link_do_pliku` (link do pliku), `ln -s /nie/istnieje wiszacy` (link wiszący), `ln -s .. link_do_katalogu`. Sprawdzisz przez `ls -l`.
3. **Jakie pola ma `struct dirent` w Linuksie?** `d_ino` (i-węzeł), `d_off` (pozycja w katalogu), `d_reclen` (długość rekordu), `d_type` (typ: `DT_REG`, `DT_DIR`, `DT_LNK`...), `d_name[256]` (nazwa). POSIX gwarantuje tylko `d_ino` i `d_name`, więc używaj tylko ich.

### Katalog roboczy (l1-2)

4. **Nieistniejący katalog, brak praw, ścieżki względne/bezwzględne?** Dla nieistniejącego `chdir` zwraca błąd `ENOENT`, a brak praw `x` daje `EACCES`; w obu przypadkach `ERR` kończy program. Brak praw `r` przy obecnym `x` wywoła błąd dopiero w `opendir`. Ścieżki względne i bezwzględne działają, bo program po każdym katalogu wraca do startowego.
5. **Dlaczego program zapamiętuje katalog roboczy?** Bez powrotu druga ścieżka względna byłaby liczona od pierwszego katalogu (`dir1/dir2/dir3` zamiast `dir2/dir3`).
6. **Czy program musi „wrócić” do katalogu startowego?** Nie dla powłoki: katalog roboczy jest własnością procesu, a powłoka po zakończeniu programu jest tam, gdzie była. Powrót jest potrzebny tylko po to, żeby kolejne ścieżki względne z `argv` były poprawne.
7. **Który błąd można obsłużyć inaczej niż wyjściem?** Błąd `chdir(argv[i])`: wypisz komunikat i `continue`, przejdź do następnego katalogu.

### Tworzenie pliku (prog12)

8. **Jaką maskę tworzy `~perms & 0777`?** Bity uprawnień, których NIE ma w `perms`, ograniczone do 9 bitów. Dla `perms = 0640` to `0137`. Jako `umask` zabiera dokładnie te bity, więc `fopen` (0666) daje `0666 & ~0137 = 0640`.
9. **Jak działa losowanie znaków?** Losowa jest tylko POZYCJA (`rand() % size`). Same litery idą po kolei: `'A' + (i % 26)` daje A, B, C... Z, A, B...
10. **Dlaczego rozmiar pliku bywa inny niż `-s`?** Rozmiar to ostatnia zapisana pozycja + 1. Przy małym `-s` zapisów jest bardzo mało (10% z 10 to 1 zapis), więc najdalsza pozycja jest losowa i plik wychodzi krótszy. Przy dużych rozmiarach znaczenie ma `RAND_MAX`: gdzie wynosi 32767, `rand() % size` nigdy nie przekroczy 32 KB i pliki > 64 KB są obcięte. Do tego dochodzi nierównomierność `%`.
11. **Jak wymusić dokładny rozmiar?** Po pętli zapisz bajt na pozycji `size - 1` (`fseek(f, size - 1, SEEK_SET); fputc(0, f);`), przepis 16.7. Alternatywa: `ftruncate(fileno(f), size)`.
12. **Dlaczego przy `unlink` ignorujemy jeden błąd?** `ENOENT` oznacza, że pliku nie było. Celem jest „brak starego pliku”, więc to nie jest błąd.

### Buforowanie (prog13)

13. **Uruchomienie w terminalu.** Liczby 0-14 pojawiają się co sekundę, bo stdout do terminala jest buforowany liniowo, a każda linia ma `\n`.
14. **Przekierowanie do pliku (`> out.txt`), podgląd w trakcie i po Ctrl+C.** W trakcie plik jest pusty: do pliku stdout jest buforowany całkowicie. Po Ctrl+C plik zostaje pusty, bo bufor nie został wypchnięty. Gdy program skończy się normalnie, wszystko pojawi się naraz.
15. **Bez `\n` w terminalu.** Nic nie widać do końca programu: bufor liniowy czeka na `\n`.
16. **To samo z `fprintf(stderr, ...)`.** Wszystko pojawia się od razu we wszystkich trzech wariantach (też w pliku), bo stderr nie jest buforowany. Przekierowanie stderr to `2>` (w tutorialu literówka `>2`): `./prog 2> err.txt`.

## 18. Pułapki, checklista, terminal, `man`

### 18.1 Najczęstsze błędy

| Błąd | Skutek | Jak poprawnie |
| --- | --- | --- |
| brak `#define _XOPEN_SOURCE 700` na górze | „implicit declaration” `getline`/`strdup`/`nftw` | pierwsza linia pliku |
| `errno` nie zerowane przed `readdir` | fałszywy błąd albo przeoczony błąd | `errno = 0;` przed każdym `readdir` |
| `lstat(entry->d_name)` dla katalogu innego niż `.` | `ENOENT` | `lstat("katalog/nazwa")` albo `chdir` |
| brak pominięcia `.` i `..` | złe liczniki, nieskończona rekurencja | `strcmp` na początku pętli |
| `stat` zamiast `lstat` | linki liczone jako pliki, pętle w rekurencji | `lstat` / `FTW_PHYS` |
| `a == b` na napisach | porównanie adresów | `strcmp(a, b) == 0` |
| `malloc(strlen(s))` | brak miejsca na `'\0'` | `malloc(strlen(s) + 1)` |
| `sizeof(ptr)` jako rozmiar napisu | 8 zamiast długości | `strlen` lub przekazany rozmiar |
| `strncpy` bez ręcznego `'\0'` | napis bez końca | `buf[n-1] = '\0'` albo `snprintf` |
| `printf(argv[i])` | dziura format string | `printf("%s", argv[i])` |
| brak `free` / `fclose` / `closedir` | wyciek, odjęte punkty | każdy zasób zwolnij |
| `free` na `optarg`, `getenv`, `readdir` | crash | zwalniaj tylko swoją pamięć |
| symlink z ścieżką względną od katalogu roboczego | link wiszący | ścieżka liczona od katalogu LINKU (`../../...`) |
| `fopen` + chcesz prawa z `x` | niemożliwe | `open(..., O_CREAT, 0755)` albo `chmod` |
| debug przez `printf` bez `\n` | nic nie widać | `fprintf(stderr, ...)` |
| `ERR` przy błędzie bez `errno` | „Success” w komunikacie | `fprintf(stderr, ...)` + `exit` |

### 18.2 Checklista przed oddaniem

- [ ] Kompiluje się z `-Wall -Wextra` bez ostrzeżeń.
- [ ] Każde wywołanie systemowe ma sprawdzony wynik (`if (...) ERR(...)`).
- [ ] Każdy `malloc`/`strdup`/`getline` ma swoje `free`; każdy `fopen`/`opendir`/`open` ma `fclose`/`closedir`/`close`.
- [ ] Złe argumenty (nieznana opcja, brak wymaganej, nadmiarowe) wypisują `usage` i kończą się `EXIT_FAILURE`.
- [ ] Komunikaty błędów idą na `stderr`, wyniki na `stdout`.
- [ ] Pominięte `.` i `..`; użyte `lstat`, gdzie liczą się linki.
- [ ] Sprawdzone przypadki brzegowe: nieistniejący plik, brak praw, pusty katalog, plik bez końcowego `\n`, ponowne uruchomienie.
- [ ] Program uruchomiony z sanitizerem lub `valgrind`: zero wycieków.

### 18.3 Komendy terminala do testów

```bash
ls -la                      # wszystkie pliki z prawami, rozmiarem, celem linków
ls -l index/by-title        # cele symlinków: nazwa -> ../../library/...
stat plik                   # pełne metadane (rozmiar, bloki, prawa, i-węzeł)
mkdir -p a/b/c              # drzewo katalogów do testów
touch a/plik.txt            # pusty plik
echo "title:X" > a/plik     # plik z treścią
printf 'a:1\nb:2' > f       # bez końcowego \n
head -c 1000 /dev/urandom > big.bin   # plik o rozmiarze 1000 B
ln -s ../a/plik.txt b/link  # symlink
readlink b/link             # na co wskazuje
chmod 000 katalog           # brak dostępu (test błędów); przywróć: chmod 755 katalog
rm -rf index                # usuń wynik poprzedniego uruchomienia
echo $?                     # kod wyjścia ostatniego programu
./prog > out.txt 2> err.txt # osobno stdout i stderr
export L1_OUTPUTFILE=out.txt
xxd db.bin | head           # podgląd pliku binarnego
tree                        # drzewo katalogów (jeśli zainstalowane)
```

### 18.4 Strony `man` do przeczytania

| Temat | Strony |
| --- | --- |
| Katalogi | `man 3p fdopendir` (opendir), `man 3p readdir`, `man 3p closedir`, `man 0p dirent.h` |
| Metadane | `man 3p fstatat` (stat, lstat), `man 0p sys_stat.h`, `man 7 inode` |
| Katalog roboczy | `man 3p getcwd`, `man 3p chdir` |
| Strumienie | `man 3p fopen`, `man 3p fclose`, `man 3p fseek`, `man 3p fread`, `man 3p getline` |
| Pliki i prawa | `man 3p mkdir`, `man 3p unlink`, `man 3p rename`, `man 3p chmod`, `man 3p access`, `man 3p umask` |
| Dowiązania | `man 3p symlink`, `man 3p readlink`, `man 3p link`, `man 1 ln` |
| Drzewo | `man 3p nftw` |
| Deskryptory | `man 3p open`, `man 3p read`, `man 3p write`, `man 3p readv`, `man 3p lseek` |
| Inne | `man 3p getopt`, `man 3p strtol`, `man 3p getenv`, `man 3p rand`, `man 3p errno`, `man 3 string` |
