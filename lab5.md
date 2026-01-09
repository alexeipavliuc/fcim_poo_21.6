# Лабораторная работа 5
## Обобщенное программирование, шаблоны функций, статический полиморфизм
### Задание
Необходимо написать код, который иллюстрирует использование шаблонов в языке С++. С помощью ключевых слов auto и template нужно написать обобщенные функции, которые принимают диапазоны любой длины, объекты любого типа, произвольное количество аргументов.

#include <iostream>
#include <initializer_list>
#include <cassert>
#include <algorithm>
#include <numeric>
#include <iterator>
#include <vector>
#include <cstddef>
#include <locale.h>
#include <string>
#include <functional>

//  ШАБЛОННАЯ ФУНКЦИЯ print_range
//  Печатает элементы диапазона [first, last) с текстовой меткой.
//  Используется универсальный подход: функция работает с любыми итераторами.
template <typename It>
void print_range(It first, It last, const std::string& label) {
    std::cout << label;   // Выводим метку перед элементами

    // Проходим циклом по диапазону, используя итераторы.
    for (auto it = first; it != last; ++it) {
        std::cout << *it << ' ';   // Разыменовываем итератор и печатаем значение
    }

    std::cout << '\n';             // Завершаем строку
}

//
//  ШАБЛОННАЯ ФУНКЦИЯ range_sum
//  Суммирует элементы диапазона. Принимает любые итераторы,
//  использует iterator_traits для определения типа значений.

template <typename It>
auto range_sum(It first, It last) {
    // iterator_traits позволяет определить value_type итератора —
    // то есть реальный тип элементов последовательности.
    using value_type = typename std::iterator_traits<It>::value_type;

    // Инициализируем сумму нулевым элементом типа value_type
    value_type result{};

    // Проходим по элементам диапазона и накапливаем сумму
    for (; first != last; ++first) {
        result += *first;      // Разыменовываем итератор и добавляем значение
    }
    return result;             // Возвращаем итоговую сумму
}

//
//  ВАРИАДИК ШАБЛОН — variadic_sum
//  Складывает произвольное количество аргументов.
//  Используется fold-expression из C++17: (args + ...)
template <typename... Args>
auto variadic_sum(Args... args) {
    return (args + ...);   // Свертка: (((arg1 + arg2) + arg3) + ...)
}

//
//  for_each_generic
//  Универсальная обобщённая функция для применения переданного
//  функторного объекта (лямбды, функции, функционального объекта)
//  ко всем элементам контейнера, поддерживающего range-based for.
template <typename Range, typename Func>
void for_each_generic(Range& range, Func f) {
    // range-based for автоматически использует begin()/end() контейнера
    for (auto& x : range) {
        f(x);    // Применяем переданную функцию к каждому элементу
    }
}

//
//  ШАБЛОННЫЙ КЛАСС example<T, U>
//  — Собственный динамический контейнер наподобие std::vector<T>
//  — template параметр U отвечает за компаратор (по умолчанию std::less<T>)
//  — Внутри используется динамический массив (new[]), управление памятью вручную
//  — Поддерживает итераторы случайного доступа (RandomAccessIterator)
template <typename T, typename U = std::less<T>>
class example {
private:
    T* data_ = nullptr;         // Указатель на динамически выделенный массив элементов
    std::size_t size_ = 0;      // Текущее количество элементов
    std::size_t capacity_ = 0;  // Выделенная ёмкость (capacity) массива
    //  ФУНКЦИЯ reallocate
    //  Перевыделяет память при необходимости увеличения capacity.
    //  Выполняет "vector-like growth": копирование элементов в новый массив.
    void reallocate(std::size_t new_cap) {
        // Если новая емкость не превышает текущую, пересоздание не требуется
        if (new_cap <= capacity_) return;

        // Выделяем новый массив под new_cap элементов
        T* new_data = new T[new_cap];

        // Копируем существующие элементы в новый массив
        for (std::size_t i = 0; i < size_; ++i) {
            new_data[i] = data_[i];
        }

        // Освобождаем старый массив
        delete[] data_;

        // Перенаправляем указатель на новый массив
        data_ = new_data;

        // Обновляем текущую емкость
        capacity_ = new_cap;
    }

public:
    //                            КЛАСС iterator
    //  Этот итератор реализует поведение RandomAccessIterator — самого мощного 
    //  типа итераторов в STL. Это означает:
    //
    //   • поддерживает перемещение вперёд (++it, it++) и назад (--it, it--)
    //   • поддерживает арифметику указателей (it + n, it - n, it2 - it1)
    //   • разыменование *it возвращает ссылку на элемент массива
    //   • operator-> позволяет обращаться к членам объекта (если T — структура/класс)
    //   • поддерживает сравнение (== и !=)
    //
    //  Итератор хранит обычный сырой указатель T*, и _полностью_
    //  моделирует поведение "указателя на элемент динамического массива".
    //
    //  Благодаря этому example<T> совместим со всеми алгоритмами STL:
    //      std::sort, std::find, std::accumulate, std::count_if, std::reverse
    class iterator {
    private:
        T* ptr;  
        // ptr — реальный указатель на текущий элемент контейнера.
        // Итератор НЕ хранит индексы, НЕ хранит ссылку на контейнер —
        // он работает как обычный указатель, что делает его быстрым
        // и полностью совместимым с архитектурой STL.
    public:
        // Эти typedef'ы — формальное описание поведения итератора.
        // Они нужны STL, чтобы определить, какие операции разрешены:
        //  iterator_category — категория итератора (RandomAccess)
        //  value_type        — тип элементов (T)
        //  difference_type   — тип для расстояний между итераторами (ptrdiff_t)
        //  pointer           — тип, представляющий указатель на элемент
        //  reference         — тип ссылки на элемент
        //
        // Без этих alias STL-алгоритмы не смогут использовать итератор.
        using iterator_category = std::random_access_iterator_tag;
        using value_type        = T;
        using difference_type   = std::ptrdiff_t;
        using pointer           = T*;
        using reference         = T&;

        // Конструктор итератора.
        // Принимает указатель на элемент массива. Если p = nullptr,
        // то итератор считается "пустым" (не указывает на элемент).
    
        iterator(T* p = nullptr) : ptr(p) {}

        // Копирующий конструктор.
        // Итератор копируется просто как копия указателя.
       
        iterator(const iterator&) = default;

        
        // ПРЕФИКСНЫЙ ИНКРЕМЕНТ (++it)
        // 1) Сначала перемещает итератор на следующий элемент
        // 2) Возвращает *ссылку* на изменённый объект
        //
        // Префиксная форма более эффективная, потому что не создаёт
        // временный объект. Алгоритмы STL часто используют именно ++it.
        
        iterator& operator++() { 
            ++ptr;     // Указатель двигается на sizeof(T) байт вперёд
            return *this;
        }

        
        // ПОСТФИКСНЫЙ ИНКРЕМЕНТ (it++)
        
        // 1) Создаёт временную копию текущего итератора
        // 2) Сдвигает указатель на следующий элемент
        // 3) Возвращает прежнее состояние
        
        // Это нужно в ситуациях типа:
        //      while (it++ != end)
        
        iterator operator++(int) { 
            iterator tmp(*this);  // Сохраняем старое состояние
            ++ptr;                // Двигаем указатель вперёд
            return tmp;           // Возвращаем прошлый итератор
        }

        
        // ПРЕФИКСНЫЙ ДЕКРЕМЕНТ (--it)
        // Перемещает итератор назад (к предыдущему элементу) 
        // и возвращает ссылку на себя.
        
        // Работает аналогично указателю: --ptr означает ptr -= 1.
        
        iterator& operator--() { 
            --ptr;
            return *this;
        }
        
        // ПОСТФИКСНЫЙ ДЕКРЕМЕНТ (it--)
        // Как и в постфиксном ++, сначала сохраняет копию.
       
        iterator operator--(int) { 
            iterator tmp(*this);
            --ptr;
            return tmp;
        }

        
        // ОПЕРАТОР РАЗЫМЕНОВАНИЯ *it
        
        // Возвращает ссылку на текущий элемент, на который указывает итератор.
        // Если ptr == nullptr — неопределённое поведение, как и у обычного указателя.
       
        reference operator*() const { 
            return *ptr; 
        }

        
        // ОПЕРАТОР СТРЕЛКА it->member
        // Позволяет обращаться к членам структур/классов:
        //      iterator it = ...;
        //      it->field;     // то же самое что (*it).field
        
        // Возвращает "сырой" указатель ptr.
        
        pointer operator->() const { 
            return ptr; 
        }

        // СРАВНЕНИЕ ИТЕРАТОРОВ

        // Два итератора равны, если указывают на один и тот же элемент.
        // Никакой дополнительной логики не нужно, достаточно сравнить ptr.
        
        friend bool operator==(const iterator& a, const iterator& b) { 
            return a.ptr == b.ptr; 
        }

        friend bool operator!=(const iterator& a, const iterator& b) { 
            return a.ptr != b.ptr; 
        }

        
        // РАЗНОСТЬ ИТЕРАТОРОВ (a - b)

        // Для RandomAccessIterator это обязательная операция.
        // Возвращает расстояние между указателями в элементах (НЕ в байтах!).
        
        // Например:
        //      it2 - it1 == 5   означает, что it2 на 5 элементов правее.
        //
        // Реализуется просто как разность указателей: ptrdiff_t.
        
        friend difference_type operator-(const iterator& a, const iterator& b) {
            return a.ptr - b.ptr;
        }

        
        
        // ОПЕРАЦИИ ПЕРЕМЕЩЕНИЯ ИТЕРАТОРА НА n ПОЗИЦИЙ
        
        // it += n  — сдвинуть вправо на n элементов
        // it -= n  — сдвинуть влево на n элементов
        //
        // Соответствует поведению указателя: ptr += n.
        iterator& operator+=(difference_type n) {
            ptr += n;
            return *this;
        }

        iterator& operator-=(difference_type n) {
            ptr -= n;
            return *this;
        }

        
        // ОПЕРАЦИИ СОЗДАНИЯ НОВОГО ИТЕРАТОРА СДВИНУТОГО НА n ЭЛЕМЕНТОВ
        
        // it + n, n + it, it - n
       
        // Важно: эти операции НЕ изменяют исходный итератор.

        friend iterator operator+(const iterator& it, difference_type n) {
            return iterator(it.ptr + n);
        }

        friend iterator operator+(difference_type n, const iterator& it) {
            return iterator(it.ptr + n);
        }

        friend iterator operator-(const iterator& it, difference_type n) {
            return iterator(it.ptr - n);
        }
    };
    
    //                          КЛАСС reverse_iterator
   
    //  Это итератор, который движется по элементам контейнера в обратном порядке.
    //  Он моделирует RandomAccessIterator, но работает "зеркально":
    //        rbegin() = iterator, указывающий на КОНЕЦ прямой последовательности
    //        rend()   = iterator, указывающий на НАЧАЛО последовательности
    //  ВАЖНОЕ ПОВЕДЕНИЕ:
    //     • обычный iterator хранит ptr → на текущий элемент
    //     • reverse_iterator хранит ptr → на позицию ПОСЛЕ текущего элемента
    //  Это требует, чтобы operator* разыменовывал (ptr - 1).
    //  Такое поведение полностью повторяет std::reverse_iterator.
    class reverse_iterator {
    private:
        T* ptr;  
        // ptr указывает на ЭЛЕМЕНТ СПРАВА от того, который нужно вернуть.
        // Если прямой итератор указывает на data_[i],
        // то reverse_iterator хранит data_[i + 1].
        
        // Благодаря такой схеме begin/end и rbegin/rend идеально согласуются.

    public:
        using iterator_category = std::random_access_iterator_tag;
        using value_type        = T;
        using difference_type   = std::ptrdiff_t;
        using pointer           = T*;
        using reference         = T&;

       
        // Конструктор итератора.
        // Просто принимает указатель ptr и сохраняет его.
        
        reverse_iterator(T* p = nullptr) : ptr(p) {}

        
        // Копирующий конструктор — тривиален.
        
        reverse_iterator(const reverse_iterator&) = default;

        
        // ПРЕФИКСНЫЙ ИНКРЕМЕНТ (++rit)
        
        //  Для прямого итератора ++ — это движение вперёд (к большим индексам).
        //  Для ОБРАТНОГО итератора ++ — это движение НАЗАД (к меньшим индексам).
       
        //     ++rit = переход к предыдущему элементу в обычном порядке.
        
        reverse_iterator& operator++() {
            --ptr;      // Двигаемся влево по массиву
            return *this;
        }

        
        // ПОСТФИКСНЫЙ ИНКРЕМЕНТ (rit++)

        // 1) сохраняем старое состояние
        // 2) сдвигаем ptr влево
        // 3) возвращаем старую версию
        
        // Полностью соответствует поведению std::reverse_iterator.
        
        reverse_iterator operator++(int) {
            reverse_iterator tmp(*this);
            --ptr;
            return tmp;
        }

        
        // ПРЕФИКСНЫЙ ДЕКРЕМЕНТ (--rit)
        
        // Теперь мы движемся в обратную сторону:
        // увеличение ptr означает продвижение вперёд в обычной последовательности.
        
        reverse_iterator& operator--() {
            ++ptr;      // Сдвигаемся вправо — движемся "вперёд" в обычном порядке
            return *this;
        }

        
        // ПОСТФИКСНЫЙ ДЕКРЕМЕНТ
        
        reverse_iterator operator--(int) {
            reverse_iterator tmp(*this);
            ++ptr;
            return tmp;
        }

        
        // ОПЕРАТОР РАЗЫМЕНОВАНИЯ *rit
        
        // reverse_iterator хранит указатель ptr на ЭЛЕМЕНТ СПРАВА,
        // поэтому реальный элемент — это *(ptr - 1)
        
        // Например:
        //     прямой итератор begin() → ptr = &data_[0]
        //     reverse_iterator rbegin() должен указывать на data_[size_-1]
        //     поэтому rbegin() хранит ptr = &data_[size_]
        
        //     *rbegin() = *(ptr - 1) = data_[size_ - 1]
        
        reference operator*() const {
            return *(ptr - 1);
        }

        
        // operator-> действует аналогично operator*
        
        // Он возвращает указатель на элемент (ptr - 1),
        // так что можно обращаться к полям структуры.
        
        pointer operator->() const {
            return (ptr - 1);
        }

        
        // Сравнение двух reverse_iterator:
        // Они равны, если указывают на одно и то же место.
        
        friend bool operator==(const reverse_iterator& a, const reverse_iterator& b) {
            return a.ptr == b.ptr;
        }

        friend bool operator!=(const reverse_iterator& a, const reverse_iterator& b) {
            return a.ptr != b.ptr;
        }
    };
public:

    // Создаёт "пустой" контейнер:
    //   • data_ = nullptr  → памяти ещё нет
    //   • size_ = 0        → элементов нет
    //   • capacity_ = 0    → емкость равна нулю
    //
    // Такое поведение идентично std::vector<T>().
   
    example() = default;



    
    //       КОНСТРУКТОР С РАЗМЕРОМ: example(std::size_t n, const T& value)
    // Создаёт контейнер с n элементами, каждый инициализирован value.
    // Шаги:
    //   1) size_ = n, capacity_ = n
    //   2) выделяем динамический массив на n элементов
    //   3) заполняем массив значением value
    // Полное соответствие поведению std::vector(size, value).
    explicit example(std::size_t n, const T& value = T{})
        : size_(n), capacity_(n)
    {
        // Выделяем память ровно под n элементов
        data_ = new T[capacity_];

        // Заполняем массив копированием значения value
        for (std::size_t i = 0; i < size_; ++i)
            data_[i] = value;
    }



    // Позволяет писать:
    //        example<int> e{1, 2, 3, 4};
    // Инициализаторный список — это объект, содержащий все переданные элементы.
    // Контейнер подстраивается под его размер.
    // Шаги:
    //   1) size_ = list.size()
    //   2) capacity_ = list.size()
    //   3) выделяем массив под capacity_
    //   4) копируем элементы из списка в массив
    example(std::initializer_list<T> list)
        : size_(list.size()), capacity_(list.size())
    {
        data_ = new T[capacity_];

        std::size_t i = 0;

        // Копируем элементы из initializer_list в массив
        for (const auto& v : list) {
            data_[i++] = v;
        }
    }



    // Реализует глубокую копию:
    //   • создаём новое хранилище
    //   • копируем значения из other.data_
    //   • никакого совместного владения памятью
    // Это поведение полностью идентично std::vector<T>.
    example(const example& other)
        : size_(other.size_), capacity_(other.capacity_)
    {
        // Выделяем массив под capacity_
        data_ = new T[capacity_];

        // Копируем каждый элемент
        for (std::size_t i = 0; i < size_; ++i)
            data_[i] = other.data_[i];
    }



    // Концепция move:
    //   • мы НЕ копируем элементы
    //   • мы забираем память其他 контейнера
    //   • other становится "пустым" объектом
    // Это избавляет от затрат на копирование большого массива.
    // ВАЖНО:
    //   example example(example&& other)
    //       забирает указатель other.data_
    //       other.data_ = nullptr
    example(example&& other) noexcept
        : data_(other.data_),
          size_(other.size_),
          capacity_(other.capacity_)
    {
        // Обнуляем старый объект, чтобы он не освободил память повторно
        other.data_ = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;
    }



    // Позволяет создать example<T> из любого диапазона:
    //     example<int> e(vec.begin(), vec.end());
    // Логика:
    //   • начинаем с пустого массива
    //   • для каждого элемента диапазона:
    //         если места нет → увеличиваем capacity в 2 раза
    //         копируем элемент
    // Это делает контейнер совместимым с итераторами любых типов.

    template <typename It>
    example(It first, It last) {
        size_ = 0;
        capacity_ = 0;
        data_ = nullptr;

        for (auto it = first; it != last; ++it) {

            // Если места не хватает — увеличиваем capacity (как std::vector)
            if (size_ == capacity_) {
                std::size_t new_cap = (capacity_ == 0 ? 4 : capacity_ * 2);
                reallocate(new_cap);
            }

            // Копируем элемент из диапазона
            data_[size_++] = *it;
        }
    }



    // Вызывается автоматически при уничтожении объекта:
    //   • Если data_ != nullptr → освобождаем динамическую память (delete[])
    //   • size_ / capacity_ не трогаем — объект всё равно исчезает
    // ВАЖНО:
    //   delete[] вызывает деструктор для КАЖДОГО элемента массива,
    //   если T — класс.
    // Это полностью аналогично поведению std::vector.
    
    ~example() {
        delete[] data_;
        // data_ становится невалидным указателем, но объект уже уничтожается
    }



    // Поведение:
    //   a = b;
    // Шаги:
    //   1) Проверяем самоприсваивание (a = a)
    //   2) Если у нас недостаточно capacity — пересоздаём массив
    //   3) Копируем элементы один за другим
    // Строгое глубокое копирование — никакого совместного владения памятью.
    example& operator=(const example& other) {
        // Проверка на присваивание объекту самому себе
        if (this != &other) {

            // Если в текущем объекте недостаточно памяти — пересоздаём массив
            if (other.size_ > capacity_) {

                delete[] data_;           // освобождаем старую память
                capacity_ = other.size_;  // новая capacity равна размеру other
                data_ = new T[capacity_]; // выделяем новый массив
            }

            // Копируем значения
            size_ = other.size_;
            for (std::size_t i = 0; i < size_; ++i)
                data_[i] = other.data_[i];
        }

        return *this;
    }



    //     a = std::move(b);
    // Шаги:
    //   1) Проверка self-move (почти невозможна, но корректна)
    //   2) Освобождаем текущую память a
    //   3) Забираем память b
    //   4) Обнуляем объект b
    // Это эквивалентно "воровству ресурсов".
    example& operator=(example&& other) noexcept {
        if (this != &other) {

            // Освобождаем текущий массив
            delete[] data_;

            // "Воруем" указатель и размеры
            data_ = other.data_;
            size_ = other.size_;
            capacity_ = other.capacity_;

            // Обнуляем другой контейнер, чтобы он не освободил память повторно
            other.data_ = nullptr;
            other.size_ = 0;
            other.capacity_ = 0;
        }

        return *this;
    }
    // Возвращает текущее количество элементов контейнера.
    // Аналогичен std::vector::size().
    std::size_t size() const {
        return size_;   // просто возвращаем значение поля
    }



    // Проверяет, пуст ли контейнер.
    // Возвращает true, если size_ == 0.
    // Аналогично std::vector::empty().
    bool empty() const {
        return size_ == 0;
    }


    // Версия для изменяемых объектов возвращает ссылку T&.
    // Мы используем assert для защиты от выхода за границы.
    //
    // Обращение происходит в стиле массива:
    //     a[i] = 10;
    // ВАЖНО: assert работает только в debug-сборках.
    T& operator[](std::size_t index) {
        assert(index < size_ && "index out of range");  // Защита от invalid index
        return data_[index];  // Возвращаем ссылку на элемент
    }



    
    // Используется, если объект example<T> является const.
    // Возвращает const T&, запрещая изменение элемента.
    const T& operator[](std::size_t index) const {
        assert(index < size_ && "index out of range");
        return data_[index];
    }



    // Проверяет, встречается ли элемент value в контейнере.
    // Аналог поведения std::vector::contains (начиная с C++23).
    // Линейный поиск O(n):
    //   • просматриваем массив от начала до конца
    //   • если нашли — возвращаем true
    bool contains(const T& value) const {
        for (std::size_t i = 0; i < size_; ++i) {

            if (data_[i] == value)     // нашли нужный элемент
                return true;
        }

        return false;  // не нашли
    }



    // Считает количество появлений значения value в массиве.
    // Логика:
    //   • проходим по каждому элементу
    //   • если равен — увеличиваем счётчик c
    // Аналог std::count из <algorithm>.
    std::size_t count(const T& value) const {
        std::size_t c = 0;

        for (std::size_t i = 0; i < size_; ++i) {

            if (data_[i] == value)
                ++c;  // увеличиваем количество найденных value
        }

        return c;
    }



    // Полностью заменяет содержимое контейнера значениями из заданного диапазона.
    // Шаги:
    //   1) size_ = 0  → очищаем контейнер
    //   2) для каждого элемента диапазона:
    //        если нет места — увеличиваем capacity в 2 раза
    //        копируем элемент
    // ВАЖНО:
    //   Это аналог std::vector::assign(first, last).
    void assign(iterator first, iterator last) {
        size_ = 0;  // очищаем текущий контейнер

        for (auto it = first; it != last; ++it) {

            // Проверяем необходимость расширения массива
            if (size_ == capacity_) {
                std::size_t new_cap = (capacity_ == 0 ? 4 : capacity_ * 2);
                reallocate(new_cap);
            }

            // Копируем текущий элемент диапазона
            data_[size_++] = *it;
        }
    }



    // Возвращают итераторы, указывающие на:
    //   begin() → первый элемент (data_)
    //   end()   → элемент, следующий после последнего (data_ + size_)
    // Это стандартный интерфейс контейнеров STL.
    iterator begin() {
        return iterator(data_);
    }

    iterator end() {
        return iterator(data_ + size_);
    }


    // Возвращают ОБРАТНЫЕ итераторы:
    //   rbegin() = reverse_iterator(data_ + size_)
    //       указывает на ПОЗИЦИЮ ПОСЛЕ последнего элемента
    //   rend()   = reverse_iterator(data_)
    //       указывает НА ПОЗИЦИЮ ПЕРЕД первым элементом
    // Такое поведение идентично std::vector::rbegin / rend.
    reverse_iterator rbegin() {
        return reverse_iterator(data_ + size_);
    }

    reverse_iterator rend() {
        return reverse_iterator(data_);
    }



    // Линейный поиск по массиву:
    //   - если нашли элемент → возвращаем iterator, указывающий на него
    //   - если не нашли → возвращаем end()
    // Аналог std::find для нашего контейнера.
    iterator find(const T& value) const {

        for (std::size_t i = 0; i < size_; ++i) {

            if (data_[i] == value) {
                // Возвращаем итератор, указывающий на позицию i
                return iterator(data_ + i);
            }
        }

        // Не нашли элемент → возвращаем end()
        return iterator(data_ + size_);
    }



    // Вставляет новый элемент в позицию pos.
    // Шаги:
    //   1) вычисляем индекс вставки (pos - begin())
    //   2) расширяем массив при необходимости
    //   3) сдвигаем все элементы справа на 1 позицию вправо
    //   4) записываем value в освободившееся место
    // Возвращает итератор на новый элемент.
    // Это аналог std::vector::insert.
    iterator insert(iterator pos, const T& value) {

        // Вычисляем индекс вставки как разницу итераторов
        std::size_t index = pos - begin();

        // Если нет места — увеличиваем capacity в 2 раза
        if (size_ == capacity_) {
            std::size_t new_cap = (capacity_ == 0 ? 4 : capacity_ * 2);
            reallocate(new_cap);
        }

    // Удаляет элемент, на который указывает итератор pos.
    // Шаги:
    //   1) вычисляем индекс удаляемого элемента
    //   2) сдвигаем элементы справа НАЛЕВО
    //   3) уменьшаем size_
    //   4) возвращаем итератор на позицию бывшего элемента
    // Аналог std::vector::erase.
    iterator erase(iterator pos) {

        std::size_t index = pos - begin();

        // Если индекс за пределами — возвращаем end(), ничего не делаем
        if (index >= size_)
            return end();

        // Сдвигаем элементы на одну позицию влево
        for (std::size_t i = index; i + 1 < size_; ++i) {
            data_[i] = data_[i + 1];
        }

        // Уменьшаем размер контейнера
        --size_;

        // Возвращаем итератор на место удалённого элемента
        return iterator(data_ + index);
    }



    // Удаляет ВСЕ элементы, равные value, в контейнере ex.
    // Возвращает количество удалённых элементов.
    // Шаги:
    //   1) идём по массиву от 0 до size_
    //   2) если элемент равен value:
    //         сдвигаем всё справа на одну позицию влево
    //         уменьшаем size_
    //         увеличиваем счётчик removed
    // Аналог std::erase (C++20).
    // ВАЖНО:
    //   метод статический — вызывается как example<int>::erase(ex, 2);
    static std::size_t erase(example<T, U>& ex, const T& value) {

        std::size_t removed = 0;  // сколько удалили
        std::size_t i = 0;        // индекс просмотра

        while (i < ex.size_) {

            // Если нашли элемент для удаления
            if (ex.data_[i] == value) {

                // Сдвигаем элементы влево
                for (std::size_t j = i; j + 1 < ex.size_; ++j) {
                    ex.data_[j] = ex.data_[j + 1];
                }

                // Уменьшаем размер
                --ex.size_;

                ++removed;   // учитываем удаление

                // i не увеличиваем — проверяем новый элемент, сдвинутый на позицию i
            }

            else {
                // если не удалили — просто идём дальше
                ++i;
            }
        }

        return removed;
    }
    
    // Два контейнера считаются равными, если:
    //   1) size_ одинаковые
    //   2) все элементы на соответствующих позициях равны
    // Линейная сложность O(n)

    friend bool operator==(const example& a, const example& b) {

        // Сначала проверяем размеры
        if (a.size_ != b.size_)
            return false;

        // Затем сравниваем поэлементно
        for (std::size_t i = 0; i < a.size_; ++i) {
            if (a.data_[i] != b.data_[i])
                return false;
        }

        // Все элементы совпали → равны
        return true;
    }



    // Логическое отрицание operator==
    friend bool operator!=(const example& a, const example& b) {
        return !(a == b);
    }



    // Лексикографическое сравнение.
    // Используется std::lexicographical_compare:
    //   сравнивает элементы как строки или массивы:
    //   [1,2,3] < [1,2,4]   → true
    //   [5, 7] < [5, 6]     → false
    //   [2,5] < [2,5,9]     → true (меньшая длина)
    //
    // Полностью соответствует поведению std::vector
    friend bool operator<(const example& a, const example& b) {
        return std::lexicographical_compare(
            a.begin(), a.end(), 
            b.begin(), b.end()
        );
    }
example<int> ex;
std::cout << ex;
    // Позволяет вывести контейнер в поток:
    //   std::cout << ex;
    // Формат вывода:
    //   { elem1 elem2 elem3 }
    // Между элементами — пробел.
    friend std::ostream& operator<<(std::ostream& os, const example& ex) {

        os << "{ ";

        // Проходим по всем элементам контейнера
        for (std::size_t i = 0; i < ex.size_; ++i) {
            os << ex.data_[i] << " ";
        }

        os << "}";

        return os; // Позволяет цепочку: cout << ex << endl;
    }
std::cin >> ex;

    // Формат работы:
    //   user вводит последовательность значений,
    //   оператор читает их и добавляет в контейнер.
    //
    // Важно: оператор читает ПОКА поток в порядке.
    friend std::istream& operator>>(std::istream& is, example& ex) {

        T value;

        // Пока пользователь вводит корректные данные
        while (is >> value) {

            // Если места нет — увеличиваем capacity
            if (ex.size_ == ex.capacity_) {
                std::size_t new_cap = (ex.capacity_ == 0 ? 4 : ex.capacity_ * 2);
                ex.reallocate(new_cap);
            }

            ex.data_[ex.size_++] = value;  // Добавляем элемент
        }

        return is;
    }
};

