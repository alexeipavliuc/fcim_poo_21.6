# Лабораторная работа 6
## Наследование
### Задание
Необходимо написать код, который иллюстрирует механизм наследования в языке С++. Используя структуры или классы нужно создать иерархию наследования и показать использование ключевых слов public, protected и private в контексте наследования. Также необходимо показать использование множественного наследования, проиллюстрировать потенциальные проблемы, с ним связанные, и способы их решения.

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

using namespace std;

// ОБОБЩЁННЫЕ ШАБЛОННЫЕ ФУНКЦИИ
// Здесь несколько универсальных функций, работающих с итераторами и контейнерами.
// Они показывают использование шаблонов, итераторов и обобщённого программирования.

// print_range
// Печатает элементы диапазона [first, last) с заданной строкой-меткой.
// Поддерживает любые итераторы: указатели, итераторы vector, list, собственного контейнера.

template <typename It>
void print_range(It first, It last, const std::string& label) {
    std::cout << label;          // Печатаем текстовую метку
    for (auto it = first; it != last; ++it) {
        std::cout << *it << ' '; // Разыменовываем итератор и печатаем значение
    }
    std::cout << '\n';
}


// range_sum
// Суммирует элементы в диапазоне [first, last).
// Использует iterator_traits для определения типа элементов (value_type).

template <typename It>
auto range_sum(It first, It last) {
    using value_type = typename std::iterator_traits<It>::value_type;
    value_type result{};               // Нулевая инициализация суммы

    for (; first != last; ++first) {
        result += *first;              // Накопление суммы
    }
    return result;
}

// variadic_sum
// Вариадический шаблон: принимает произвольное количество аргументов
// и возвращает их сумму, используя свёртку (fold expression) из C++17.
// Пример: variadic_sum(1,2,3,4) → 10.

template <typename... Args>
auto variadic_sum(Args... args) {
    return (args + ...); // (((arg1 + arg2) + arg3) + ...)
}

// for_each_generic
// Обобщённый аналог std::for_each.
// Принимает любой "диапазон" (контейнер с begin/end) и функцию (лямбду),
// и применяет её ко всем элементам с помощью range-based for.

template <typename Range, typename Func>
void for_each_generic(Range& range, Func f) {
    for (auto& x : range) {  // Перебор всех элементов контейнера
        f(x);                // Применение переданной функции к элементу
    }
}

// ШАБЛОННЫЙ КОНТЕЙНЕР example<T, U>

// Это упрощённый аналог std::vector<T>:
//   • хранит элементы в динамическом массиве (через new/delete[])
//   • умеет автоматически увеличивать capacity при добавлении элементов
//   • предоставляет итераторы случайного доступа (Random Access Iterator)
//   • поддерживает поиск, вставку, удаление, сравнение, ввод/вывод
// Параметр U (по умолчанию std::less<T>) зарезервирован для компаратора,
// но в данном коде явно не используется.

template <typename T, typename U = std::less<T>>
class example {
private:
    T* data_ = nullptr;           // Указатель на динамический массив элементов
    std::size_t size_ = 0;        // Текущее количество элементов в контейнере
    std::size_t capacity_ = 0;    // Текущее количество выделенных ячеек памяти

    // reallocate(new_cap)
    // Вспомогательная функция увеличения capacity.
    // Если new_cap > текущей capacity — создаём новый массив, копируем элементы,
    // удаляем старый массив и переназначаем указатель.
    
    // По сути — ручная реализация роста буфера, как у std::vector.
    
    void reallocate(std::size_t new_cap) {
        if (new_cap <= capacity_)
            return; // Если новая ёмкость не больше текущей — ничего не делаем

        T* new_data = new T[new_cap];         // Новый буфер

        // Копируем все существующие элементы в новый массив
        for (std::size_t i = 0; i < size_; ++i) {
            new_data[i] = data_[i];
        }

        delete[] data_;                       // Освобождаем старую память
        data_ = new_data;                     // Переназначаем указатель
        capacity_ = new_cap;                  // Обновляем ёмкость
    }

public:
    
    // ВНУТРЕННИЙ КЛАСС iterator
    
    // Итератор случайного доступа (Random Access Iterator), работающий как
    // обычный указатель T*. Поддерживает:
    //   • ++, --, +n, -n
    //   • разыменование *it, доступ через it->member
    //   • сравнение ==, !=
    //   • разность (it2 - it1)
    // Можно использовать в стандартных алгоритмах (sort, find, accumulate).
    
    class iterator {
    private:
        T* ptr;   // Указатель на текущий элемент в массиве

    public:
        using iterator_category = std::random_access_iterator_tag;
        using value_type        = T;
        using difference_type   = std::ptrdiff_t;
        using pointer           = T*;
        using reference         = T&;

        // Конструктор по указателю. По умолчанию ptr = nullptr.
        iterator(T* p = nullptr) : ptr(p) {}
        // Копирующий конструктор по умолчанию.
        iterator(const iterator&) = default;

        // Префиксный инкремент: сначала увеличиваем указатель, потом возвращаем *this.
        iterator& operator++() {
            ++ptr;
            return *this;
        }

        // Постфиксный инкремент: возвращаем копию старого значения, затем увеличиваем ptr.
        iterator operator++(int) {
            iterator tmp(*this);
            ++ptr;
            return tmp;
        }

        // Префиксный декремент.
        iterator& operator--() {
            --ptr;
            return *this;
        }

        // Постфиксный декремент.
        iterator operator--(int) {
            iterator tmp(*this);
            --ptr;
            return tmp;
        }

        // Разыменование — доступ к элементу по ссылке.
        reference operator*() const { return *ptr; }
        // Доступ к членам объекта T (если T — класс/структура).
        pointer   operator->() const { return ptr; }

        // Сравнение итераторов на равенство (сравниваем указатели).
        friend bool operator==(const iterator& a, const iterator& b) { return a.ptr == b.ptr; }
        friend bool operator!=(const iterator& a, const iterator& b) { return a.ptr != b.ptr; }

        // Разность итераторов — количество элементов между ними.
        friend difference_type operator-(const iterator& a, const iterator& b) {
            return a.ptr - b.ptr;
        }

        // Сдвиг итератора на n элементов вправо/влево.
        iterator& operator+=(difference_type n) { ptr += n; return *this; }
        iterator& operator-=(difference_type n) { ptr -= n; return *this; }

        // Возвращают новый итератор, сдвинутый на n элементов.
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

    
    // ОБРАТНЫЙ ИТЕРАТОР reverse_iterator
    
    // Движется по контейнеру в обратном порядке. Хранит указатель ptr так,
    // что разыменование делает *(ptr - 1).
    // Это классический приём, как в std::reverse_iterator.
    
    class reverse_iterator {
    private:
        T* ptr;   // Указатель на "позицию после" текущего элемента

    public:
        reverse_iterator(T* p = nullptr) : ptr(p) {}
        reverse_iterator(const reverse_iterator&) = default;

        // ++rit для обратного итератора двигает его ВЛЕВО по массиву (к началу).
        reverse_iterator& operator++() {
            --ptr;
            return *this;
        }
        reverse_iterator operator++(int) {
            reverse_iterator tmp(*this);
            --ptr;
            return tmp;
        }

        // --rit двигает его вправо (к концу в обычном порядке).
        reverse_iterator& operator--() {
            ++ptr;
            return *this;
        }
        reverse_iterator operator--(int) {
            reverse_iterator tmp(*this);
            ++ptr;
            return tmp;
        }

        // Разыменование: возвращаем элемент слева от ptr.
        T& operator*() const { return *(ptr - 1); }
        // Доступ к членам (* (ptr - 1)).member.
        T* operator->() const { return (ptr - 1); }

        friend bool operator==(const reverse_iterator& a, const reverse_iterator& b) {
            return a.ptr == b.ptr;
        }
        friend bool operator!=(const reverse_iterator& a, const reverse_iterator& b) {
            return a.ptr != b.ptr;
        }
    };

    
    // КОНСТРУКТОРЫ, ДЕСТРУКТОР И ОПЕРАТОРЫ ПРИСВАИВАНИЯ
    

    // Конструктор по умолчанию: пустой контейнер, data_ = nullptr, size_ = 0, capacity_ = 0.
    example() = default;

    // Конструктор с количеством элементов и значением по умолчанию.
    // Создаёт массив из n элементов, каждый инициализируется value.
    explicit example(std::size_t n, const T& value = T{})
        : size_(n), capacity_(n) {
        data_ = new T[capacity_];
        for (std::size_t i = 0; i < size_; ++i)
            data_[i] = value;
    }

    // Конструктор из initializer_list — позволяет писать example<int> e{1,2,3}.
    example(std::initializer_list<T> list)
        : size_(list.size()), capacity_(list.size()) {
        data_ = new T[capacity_];
        std::size_t i = 0;
        for (const auto& v : list)
            data_[i++] = v;
    }

    // Копирующий конструктор — глубокое копирование (новый массив + копия элементов).
    example(const example& other)
        : size_(other.size_), capacity_(other.capacity_) {
        data_ = new T[capacity_];
        for (std::size_t i = 0; i < size_; ++i)
            data_[i] = other.data_[i];
    }

    // Перемещающий конструктор — "ворует" ресурсы другого контейнера.
    example(example&& other) noexcept
        : data_(other.data_), size_(other.size_), capacity_(other.capacity_) {
        other.data_ = nullptr;
        other.size_ = 0;
        other.capacity_ = 0;
    }

    // Конструктор из диапазона [first, last).
    // Позволяет создавать пример из любых итераторов (например, из vector<int>).
    template <typename It>
    example(It first, It last) {
        size_ = 0;
        capacity_ = 0;
        data_ = nullptr;

        for (auto it = first; it != last; ++it) {
            if (size_ == capacity_) {
                std::size_t new_cap = capacity_ == 0 ? 4 : capacity_ * 2;
                reallocate(new_cap);
            }
            data_[size_++] = *it;
        }
    }

    // Деструктор — освобождает динамический массив.
    ~example() {
        delete[] data_;
    }

    // Оператор копирующего присваивания.
    // Если нужно — перевыделяет память, затем копирует элементы.
    example& operator=(const example& other) {
        if (this != &other) {
            if (other.size_ > capacity_) {
                delete[] data_;
                capacity_ = other.size_;
                data_ = new T[capacity_];
            }
            size_ = other.size_;
            for (std::size_t i = 0; i < size_; ++i)
                data_[i] = other.data_[i];
        }
        return *this;
    }

    // Оператор перемещающего присваивания.
    // Сначала освобождаем текущую память, затем забираем указатель other.
    example& operator=(example&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            capacity_ = other.capacity_;
            other.data_ = nullptr;
            other.size_ = 0;
            other.capacity_ = 0;
        }
        return *this;
    }

    
    // БАЗОВЫЕ МЕТОДЫ ДОСТУПА
    

    // Возвращает текущее количество элементов.
    std::size_t size() const { return size_; }

    // Проверяет, пуст ли контейнер.
    bool empty() const { return size_ == 0; }

    // Доступ к элементу по индексу с проверкой assert.
    T& operator[](std::size_t index) {
        assert(index < size_);
        return data_[index];
    }

    // Константная версия доступа по индексу.
    const T& operator[](std::size_t index) const {
        assert(index < size_);
        return data_[index];
    }

    // Проверяет наличие элемента value в контейнере (линейный поиск).
    bool contains(const T& value) const {
        for (std::size_t i = 0; i < size_; ++i)
            if (data_[i] == value) return true;
        return false;
    }

    // Подсчитывает, сколько раз значение value встречается в контейнере.
    std::size_t count(const T& value) const {
        std::size_t c = 0;
        for (std::size_t i = 0; i < size_; ++i)
            if (data_[i] == value) ++c;
        return c;
    }

    // assign(first, last) — полностью заменяет содержимое контейнера
    // элементами из диапазона [first, last).
    void assign(iterator first, iterator last) {
        size_ = 0;
        for (auto it = first; it != last; ++it) {
            if (size_ == capacity_) {
                std::size_t new_cap = capacity_ == 0 ? 4 : capacity_ * 2;
                reallocate(new_cap);
            }
            data_[size_++] = *it;
        }
    }

    // Итераторы begin()/end() — стандартный интерфейс STL.
    iterator begin() { return iterator(data_); }
    iterator end()   { return iterator(data_ + size_); }

    // Обратные итераторы rbegin()/rend() — для обхода в обратном порядке.
    reverse_iterator rbegin() { return reverse_iterator(data_ + size_); }
    reverse_iterator rend()   { return reverse_iterator(data_); }

    // Линейный поиск элемента value. Возвращает iterator или end().
    iterator find(const T& value) const {
        for (std::size_t i = 0; i < size_; ++i)
            if (data_[i] == value) return iterator(data_ + i);
        return iterator(data_ + size_);
    }

    // Вставка value в позицию pos. Сдвиг всех элементов справа вправо.
    // Возвращает итератор на вставленный элемент.
    iterator insert(iterator pos, const T& value) {
        std::size_t index = pos - begin();
        if (size_ == capacity_) {
            std::size_t new_cap = capacity_ == 0 ? 4 : capacity_ * 2;
            reallocate(new_cap);
        }
        for (std::size_t i = size_; i > index; --i)
            data_[i] = data_[i - 1];
        data_[index] = value;
        ++size_;
        return iterator(data_ + index);
    }

    // Удаляет элемент по итератору pos. Сдвигает элементы влево.
    // Возвращает итератор на позицию после удалённого элемента.
    iterator erase(iterator pos) {
        std::size_t index = pos - begin();
        if (index >= size_) return end();
        for (std::size_t i = index; i + 1 < size_; ++i)
            data_[i] = data_[i + 1];
        --size_;
        return iterator(data_ + index);
    }

    // Статический метод erase(ex, value) — удаляет ВСЕ элементы равные value.
    // Возвращает количество удалённых элементов.
    static std::size_t erase(example<T, U>& ex, const T& value) {
        std::size_t removed = 0;
        std::size_t i = 0;
        while (i < ex.size_) {
            if (ex.data_[i] == value) {
                for (std::size_t j = i; j + 1 < ex.size_; ++j)
                    ex.data_[j] = ex.data_[j + 1];
                --ex.size_;
                ++removed;
            } else {
                ++i;
            }
        }
        return removed;
    }

    // Сравнение на равенство: размеры должны совпадать и все элементы равны.
    friend bool operator==(const example& a, const example& b) {
        if (a.size_ != b.size_) return false;
        for (std::size_t i = 0; i < a.size_; ++i)
            if (a.data_[i] != b.data_[i]) return false;
        return true;
    }

    friend bool operator!=(const example& a, const example& b) {
        return !(a == b);
    }

    // Лексикографическое сравнение < — используется std::lexicographical_compare.
    friend bool operator<(const example& a, const example& b) {
        return std::lexicographical_compare(a.data_, a.data_ + a.size_,
                                            b.data_, b.data_ + b.size_);
    }

    // Вывод в поток: печатаем элементы через пробел.
    friend std::ostream& operator<<(std::ostream& os, const example& obj) {
        for (std::size_t i = 0; i < obj.size_; ++i) {
            os << obj.data_[i];
            if (i + 1 != obj.size_) os << ' ';
        }
        return os;
    }

    // Ввод из потока: читаем элементы подряд, пока поток в порядке.
    friend std::istream& operator>>(std::istream& is, example& obj) {
        T value;
        obj.size_ = 0;
        while (is >> value) {
            if (obj.size_ == obj.capacity_) {
                std::size_t new_cap = obj.capacity_ == 0 ? 4 : obj.capacity_ * 2;
                obj.reallocate(new_cap);
            }
            obj.data_[obj.size_++] = value;
        }
        return is;
    }
};

// Здесь показаны разные виды наследования: public/protected/private
// и пример решения проблемы "ромбовидного наследования" через virtual.

class Base {
protected:
    int protected_value = 10; // доступен в самом классе и в его наследниках
public:
    int public_value = 1;     // доступен отовсюду
};

// Public-наследование: public и protected члены Base сохраняют свои уровни доступа.
class PublicDerived : public Base {
public:
    void change() {
        public_value = 100;     // можно изменять public из Base
        protected_value = 200;  // можно изменять protected из Base
    }
};

// Protected-наследование: public и protected из Base становятся protected в наследнике.
class ProtectedDerived : protected Base {
public:
    void change() {
        public_value = 300;     // внутри класса доступно
        protected_value = 400;
    }
};

// Private-наследование: public и protected из Base становятся private.
class PrivateDerived : private Base {
public:
    void change() {
        public_value = 500;     // внутри класса доступно
        protected_value = 600;
    }
};

// Множественное наследование + решение "ромба" через virtual
//
//      A
//     / \
//    B   C
//     \ /
//      D
//
// Если не использовать virtual, у D будет ДВЕ копии A.
// virtual public A → у D остаётся ОДНА общая A.

class A {
public:
    int x = 10;
};

// B и C виртуально наследуют A — один общий A для всех.
class B : virtual public A {};
class C : virtual public A {};

// D наследует B и C, но из-за virtual у него только один A::x.
class D : public B, public C {
public:
    int get() { return x; } // x однозначно определяется (нет двусмысленности)
};

// ДОП. ЗАДАНИЕ — АБСТРАКТНЫЙ ИНТЕРФЕЙС
// abstract_data_t — абстрактный базовый класс (интерфейс),
// определяющий общий контракт для "контейнеров данных":
//   • empty, size, front, back, push, pop, extend.

class abstract_data_t {
public:
    virtual bool empty() const = 0;
    virtual size_t size() const = 0;
    virtual int& front() = 0;
    virtual int& back() = 0;
    virtual void push(int) = 0;
    virtual void pop() = 0;
    virtual void extend(const abstract_data_t&) = 0;
    virtual ~abstract_data_t() = 0; // чисто виртуальный деструктор
};

// Определение виртуального деструктора (обязательно, даже если он пустой).
inline abstract_data_t::~abstract_data_t() {}

// Класс example_int
// Реализация интерфейса abstract_data_t на основе контейнера example<int>.
// Показывает полиморфизм: работаем через указатель abstract_data_t*,
// но в реальности используются методы example_int и example<int>.

class example_int : public abstract_data_t {
private:
    example<int> data;  // Внутренний контейнер, который выполняет реальную работу

public:
    bool empty() const override { return data.empty(); }
    size_t size() const override { return data.size(); }

    int& front() override { return data[0]; }
    int& back() override { return data[data.size() - 1]; }

    // Добавляем элемент в конец: используем insert в конец.
    void push(int x) override {
        data.insert(data.end(), x);
    }

    // Удаляем последний элемент, если контейнер не пуст.
    void pop() override {
        if (!data.empty()) {
            data.erase(data.begin() + (data.size() - 1));
        }
    }

    // extend(other): добавляет в текущий контейнер все элементы другого.
    // Используем dynamic_cast, чтобы убедиться, что other — это example_int.
    void extend(const abstract_data_t& other) override {
        const example_int* p = dynamic_cast<const example_int*>(&other);
        // В учебном коде предполагается, что cast успешен.
        for (size_t i = 0; i < p->data.size(); i++) {
            push(p->data[i]); // Добавляем в конец каждый элемент другого контейнера
        }
    }
};

// MAIN — демонстрация всех возможностей:
//   • наследование и решение ромба
//   • использование интерфейса abstract_data_t
//   • применение нашего контейнера example<int> через example_int

int main() {
    setlocale(LC_ALL, "Russian");

    PublicDerived pd;
    pd.change(); // Меняем значения унаследованных полей из Base

    D diamond;   // Объект с ромбовидным наследованием
    cout << "Проблема ромба решена, x = " << diamond.get() << "\n";


    // Работаем через указатель на интерфейс, но объект — example_int.
    abstract_data_t* a = new example_int;
    abstract_data_t* b = new example_int;

    // В контейнер a кладём числа 1,2,3.
    a->push(1);
    a->push(2);
    a->push(3);

    // В контейнер b кладём 100, а потом расширяем b элементами a.
    b->push(100);
    b->extend(*a); // теперь в b: 100,1,2,3

    cout << "b.front() = " << b->front() << "\n";              // первый элемент b
    cout << "b.back() = " << b->back() << "\n";                // последний элемент b
    cout << "b.size() = " << b->size() << "\n";                // размер b

    // Освобождаем память, т.к. создавали через new.
    delete a;
    delete b;

    return 0;
}
