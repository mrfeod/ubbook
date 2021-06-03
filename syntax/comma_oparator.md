# Оператор запятая

Если вы начинали свое знакомство с программированием с языков Pascal или C#, то, наверное, знаете, что
в них обращение к элементам двумерного массива (а также массивов большей размерности) осуществляется перечислением индексов через запятую внутри квадратных скобок

```C#
double [,] array = new double[10, 10];
double x = array[1,1];
```

Также в записи на псевдокоде или в специализированных языках для математических вычислений (MatLab, MathCAD) часто используют именно такой или похожий (круглые скобки) способы.

В C/C++ же на каждую размерность должны быть свои квадратные скобки

```C++
double array[10][10];
double x = array[1][1];
```

Однако, написать «неправильно» нам никто не запрещает и, более того, компилятор обязан это [скомпилировать](https://godbolt.org/z/G4zhYdnd1)!

```C++
int array[5][5] = {};
std::cout << array[1,4]; // oops!
```

В комбинации с неявным приведением типов и выходами за кграницы массивов, можно наиграть себе множество неприятностей при невнимательном переносе кода.

Почему это вообще компилируется?

Все дело в операторе «запятая» (`,`). Она последовательно вычисляет оба своих аргумента и возвращает второй (правый).

```C++
int array[2][5] = {}
auto x = array[1, 4]; // Oops! Это array[4]. 
// Но для первой размерности максимальное значение = 1. Неопределенное поведение!
```

В C++20, на наше счастье, использование оператора `,` при индексировании массивов пометили как deprecated и
теперь компиляторы [сыпять предупреждениями](https://godbolt.org/z/976Gsad1o) (вы всегда можете их превратить в ошибки)

На этом можно было бы и закончить, если бы не один нюанс.

## Перегрузки оператора `,`

Запятую можно перегрузить. И посеять немного хаоса.

```C++
return f1(), f2(), f3(); 
```
Если `,` не перегружена, стандарт гарантирует, что функции будут вызваны последовательно.
Если же тут вызывается перегруженная запятая, то до C++17 такой гарантии нет.

В случае встроенной запятой тут гарантируется, что тип результата совпадает с последним аргументов в цепочке.
Если же оператор перегружен — тип может быть [каким угодно](https://godbolt.org/z/qW5Gsfcbs).

```C++
auto test() {
    return f1(), f2(), f3();
}

int main() {
    test();
    static_assert(!std::is_same_v<decltype(f3()), int>);
    static_assert(std::is_same_v<decltype(test()), int>); // ??!
    return 0;
}
```

Запятой часто пользуются в различных шаблонах, чтобы раскрывать пачки аргументов произвольной длины, или чтобы проверять несколько условий, триггерящих SFINAE.

Из-за потенциальной возможности влететь в перегруженную запятую, в выражениях с ней авторы библиотек
прибегают к касту каждого аргумента к `void` — перегрузку, принимающую `void` невозможно написать.

```C++
template <class... F>
void invoke_all(F&&... f) {
    (static_cast<void>(f()), ...);
}

int main() {
    invoke_all([]{
        std::cout << "hello!\n";
    },
    []{
        std::cout << "World!\n";
    });
    return 0;
}
```

Зачем вообще может понадобиться перегружать запятую? 

Может быть, для какого-нибудь DSL (domain-specific language).

Или вдруг вам все-таки захочется сделать так, чтоб индексация через запятую [работала](https://godbolt.org/z/bjjrr6nd3).

```C++

struct Index { size_t idx; };

template <size_t N>
struct MultiIndex : std::array<Index, N> {};

template <size_t N, size_t M>
auto operator , (MultiIndex<N> i1, MultiIndex<M> i2) { ... }

template <size_t M>
auto operator , (Index i1, MultiIndex<M> i2) { ... }

template <size_t N>
auto operator , (MultiIndex<N> i1, Index i2) { ... }

auto operator , (Index i1, Index i2) { ... }

Index operator "" _i (unsigned long long x) {
    return Index { static_cast<size_t>(x) };
}

template <class T, size_t N, size_t M>
struct Array2D {
    T arr[N][M];

    T& operator [] (MultiIndex<2> idx) {
        return arr[idx[0].idx][idx[1].idx];
    }
};

int main() {
    Array2D<int, 5, 6> arr;

    arr[1_i, 2_i] = 5;
    std::cout << arr[1_i, 2_i]; // Ok
    std::cout << arr[1_i, 2_i, 3_i]; // Compilation error
}

```