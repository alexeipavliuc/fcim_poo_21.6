# Лабораторная работа 7
## Обработка ошибок, исключения
### Задание
Необходимо написать код, который иллюстрирует использование исключений для обработки исключительных ситуаций. Используя ключевые слова throw, try и catch, нужно обработать несколько разных ислючений. Также нужно показать эффект исключений разных типов в рамках одного блока try-catch и восстановление программы после обработки исключения.

Далее нужно создать свой тип исключений -- так, чтобы его обработка гармонично сочеталась с обработкой встроенных типов ислючений.

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
#include <concepts>
#include <type_traits>

using namespace std;

    //Эта часть демонстрирует, как создавать свои типы исключений в C++.
    
    //- Наследуемся от std::exception.
   // - Храним сообщение об ошибке в std::string.
    //- Переопределяем метод what() для вывода подробного текста.
    
    //Зачем это нужно?
    //Чтобы контейнер example<T> мог бросать понятные ошибки,
    //а не стандартные стандартные "std::out_of_range" или "assert failed".
    
class example_exception : public std::exception {
    std::string msg;    // Храним сообщение об ошибке
public:
    explicit example_exception(std::string m) : msg(std::move(m)) {}

    // what() должен возвращать const char*
    const char* what() const noexcept override { return msg.c_str(); }
};



    //ОБОБЩЁННЫЕ ФУНКЦИИ 

    //Здесь мы показываем работу с итераторами, шаблонами, variadic-шаблонами.
    
    //Эти функции универсальны: они работают с любыми контейнерами,
    //которые поддерживают необходимые операции.


// Печать диапазона с любыми итераторами
template <typename It>
void print_range(It first, It last, const std::string& label) {
    std::cout << label;
    for (auto it = first; it != last; ++it) {
        std::cout << *it << ' ';
    }
    std::cout << '\n';
}

// Суммирование диапазона с помощью iterator_traits
template <typename It>
auto range_sum(It first, It last) {
    using value_type = typename std::iterator_traits<It>::value_type;
    value_type result{};
    for (; first != last; ++first) result += *first;
    return result;
}

// Variadic-fold expression — суммирование аргументов
template <typename... Args>
auto variadic_sum(Args... args) {
    return (args + ...);
}

// Универсальная версия std::for_each
template <typename Range, typename Func>
void for_each_generic(Range& range, Func f) {
    for (auto& x : range) f(x);
}


   //КОНЦЕПЦИИ (C++20) 
    
    //Они позволяют задать ограничения на типы шаблонов, 
    //чтобы шаблонный код не принимал "что угодно".

template <typename T>
concept index_accessible = requires(T a, size_t i) {
    { a.size() } -> std::same_as<size_t>;
    { a[i] };
};

template <typename T>
concept pushable = requires(T a, int x) {
    { a.insert(a.end(), x) };
};

template <typename T>
concept poppable = requires(T a) {
    { a.size() } -> std::same_as<size_t>;
    { a.erase(a.begin()) };
};



//КОНТЕЙНЕР example<T> с ДОПОЛНЕННОЙ ФУНКЦИОНАЛЬНОСТЬЮ
    
    //Этот контейнер:
    //- Поддерживает динамическое выделение памяти
    //- Имеет forward- и reverse-итераторы
    //- Поддерживает insert(), erase(), begin(), end()
    //- Имеет своё исключение при обращении за границы
    //- Использует концепции
    //- Применяется через интерфейс abstract_data_t

template <typename T, typename U = std::less<T>>
class example {
private:
    T* data_ = nullptr;       // динамический массив
    size_t size_ = 0;         // текущее число элементов
    size_t capacity_ = 0;     // выделенная ёмкость

        //reallocate() — ключевой механизм увеличения памяти.
        //Если new_cap > capacity, выделяем новый массив и 
        //копируем туда старые данные.
    
    void reallocate(size_t new_cap) {
        if (new_cap <= capacity_) return;

        T* new_data = new T[new_cap];
        for (size_t i = 0; i < size_; ++i) new_data[i] = data_[i];

        delete[] data_;
        data_ = new_data;
        capacity_ = new_cap;
    }

public:

        //at(index) — безопасный доступ к элементу
        //Если index >= size_, бросается example_exception.
    
    T& at(size_t index) {
        if (index >= size_) {
            throw example_exception("Ошибка: выход за границы example<T>::at()");
        }
        return data_[index];
    }


    
        //ПРЯМОЙ ИТЕРАТОР
        //Упрощённая модель random_access_iterator.
    class iterator {
        T* ptr;
    public:
        iterator(T* p = nullptr) : ptr(p) {}

        iterator& operator++() { ++ptr; return *this; }
        iterator operator++(int) { iterator t(*this); ++ptr; return t; }

        iterator& operator--() { --ptr; return *this; }
        iterator operator--(int) { iterator t(*this); --ptr; return t; }

        T& operator*() const { return *ptr; }

        friend bool operator==(const iterator& a, const iterator& b) { return a.ptr == b.ptr; }
        friend bool operator!=(const iterator& a, const iterator& b) { return a.ptr != b.ptr; }

        friend ptrdiff_t operator-(const iterator& a, const iterator& b) { return a.ptr - b.ptr; }

        iterator operator+(ptrdiff_t n) const { return iterator(ptr + n); }
    };


        //ОБРАТНЫЙ ИТЕРАТОР
        //Работает зеркально: ++ движет влево.

    class reverse_iterator {
        T* ptr;
    public:
        reverse_iterator(T* p = nullptr) : ptr(p) {}

        reverse_iterator& operator++() { --ptr; return *this; }
        T& operator*() const { return *(ptr - 1); }

        friend bool operator!=(const reverse_iterator& a, const reverse_iterator& b) {
            return a.ptr != b.ptr;
        }
    };


    
       // КОНСТРУКТОРЫ контейнера example<T>

    example() = default;

    explicit example(size_t n, const T& value = T{})
        : size_(n), capacity_(n)
    {
        data_ = new T[capacity_];
        for (size_t i = 0; i < size_; ++i) data_[i] = value;
    }

    example(initializer_list<T> list)
        : size_(list.size()), capacity_(list.size())
    {
        data_ = new T[capacity_];
        size_t i = 0;
        for (const auto& v : list) data_[i++] = v;
    }

    ~example() {
        delete[] data_;
    }


        //БАЗОВЫЕ МЕТОДЫ

    size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }

    T& operator[](size_t index) {
        assert(index < size_);
        return data_[index];
    }

    iterator begin() { return iterator(data_); }
    iterator end() { return iterator(data_ + size_); }

    reverse_iterator rbegin() { return reverse_iterator(data_ + size_); }
    reverse_iterator rend() { return reverse_iterator(data_); }


        //ВСТАВКА (insert)

    iterator insert(iterator pos, const T& value) {
        size_t index = pos - begin();

        if (size_ == capacity_) {
            size_t new_cap = capacity_ == 0 ? 4 : capacity_ * 2;
            reallocate(new_cap);
        }

        for (size_t i = size_; i > index; --i)
            data_[i] = data_[i - 1];

        data_[index] = value;
        ++size_;
        return iterator(data_ + index);
    }


    
       // УДАЛЕНИЕ (erase)

    iterator erase(iterator pos) {
        size_t index = pos - begin();
        if (index >= size_) return end();

        for (size_t i = index; i + 1 < size_; ++i)
            data_[i] = data_[i + 1];

        --size_;
        return iterator(data_ + index);
    }
};

//lab6 — НАСЛЕДОВАНИЕ

        //Здесь показаны:
       // - public наследование
        //- protected наследование
       // - private наследование
        //- виртуальное наследование (решение “ромба”)

class Base {
protected:
    int protected_value = 10;
public:
    int public_value = 1;
};

class PublicDerived : public Base {
public:
    void change() {
        public_value = 100;
        protected_value = 200;
    }
};

// Ромб наследования
class A { public: int x = 10; };
class B : virtual public A {};
class C : virtual public A {};
class D : public B, public C {
public:
    int get() { return x; }
};

       // lab6 — АБСТРАКТНЫЙ ИНТЕРФЕЙС abstract_data_t

        //Интерфейс задаёт:
       // - empty()
        //- size()
       // - front(), back()
        //- push(), pop()
       // - extend()
        //- виртуальный деструктор

        example_int — адаптер example<int> под этот интерфейс.


class abstract_data_t {
public:
    virtual bool empty() const = 0;
    virtual size_t size() const = 0;

    virtual int& front() = 0;
    virtual int& back() = 0;

    virtual void push(int) = 0;
    virtual void pop() = 0;

    virtual void extend(const abstract_data_t&) = 0;

    virtual ~abstract_data_t() = 0;
};

inline abstract_data_t::~abstract_data_t() {}


class example_int : public abstract_data_t {
private:
    example<int> data;

public:

    bool empty() const override { return data.empty(); }
    size_t size() const override { return data.size(); }

    int& front() override {
        if (data.empty()) throw example_exception("front(): пустой контейнер!");
        return data[0];
    }

    int& back() override {
        if (data.empty()) throw example_exception("back(): пустой контейнер!");
        return data[data.size() - 1];
    }

    void push(int x) override {
        data.insert(data.end(), x);
    }

    void pop() override {
        if (data.empty()) throw example_exception("pop(): невозможно удалить из пустого контейнера!");
        data.erase(data.begin() + (data.size() - 1));
    }

    void extend(const abstract_data_t& other) override {
        const example_int* p = dynamic_cast<const example_int*>(&other);
        for (size_t i = 0; i < p->data.size(); i++)
            push(p->data[i]);
    }
};


    //lab7 — ОБРАБОТКА ИСКЛЮЧЕНИЙ

   // В этой функции мы специально бросаем разные исключения, 
    //чтобы показать порядок их перехвата:
    //- сначала example_exception
   // - затем std::exception

void demonstrate_exceptions() {
    try {
        cout << "\n=== Демонстрация исключений ===\n";

        throw example_exception("Моё пользовательское исключение!");
        throw std::runtime_error("runtime_error!");
        throw 123;
    }
    catch (const example_exception& e) {
        cout << "[example_exception] " << e.what() << endl;
    }
    catch (const std::exception& e) {
        cout << "[std::exception] " << e.what() << endl;
    }
    catch (...) {
        cout << "[неизвестное исключение]\n";
    }

    cout << "Программа восстановила работу после исключений.\n";
}


   // MAIN — демонстрация всех возможностей:

    //- обработка исключений (lab7)
    //- работа через интерфейс (lab6)
    //- наследование и виртуальное наследование (“ромб”)

int main() {
    setlocale(LC_ALL, "Russian");

    demonstrate_exceptions();

    abstract_data_t* a = new example_int;
    try {
        a->pop(); // Ошибка: контейнер пуст → бросается example_exception
    }
    catch (const example_exception& e) {
        cout << "Обработано: " << e.what() << endl;
    }
    delete a;

    D diamond;
    cout << "Ромб: x = " << diamond.get() << "\n";

    return 0;
}


