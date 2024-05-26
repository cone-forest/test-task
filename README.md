# Тестовое задание
### Вопрос 1
```cpp
bool isEven(int value) {
    return value & 1 == 0;
}
```
- Данная релизация использует побитовое и с единицей.
- Если число четное, то в его разбиении на степени двойки не будет единицы.
- Данная реализация использует потенциально более эффективные операции - "побитовое и" и сравнение с 0, при этом функция менее читаемая (несмотря на говорящее название).
- Стоит обратить внимание, что компилятор вероятно соптимизирует изначальную функцию до аналогичного вида.

### Вопрос 2
#### Реализация №1 (статический массив)
```cpp
#include <array>
#include <optional>

namespace mr {
  template <typename T, std::size_t S>
    struct DynamicRingBuffer {
    private:
      std::size_t _size;
      std::size_t _head;
      std::size_t _tail;
      std::array<T, S> _data;

    public:
      // default constructor
      constexpr DynamicRingBuffer() noexcept = default;
      // copy constructor
      constexpr DynamicRingBuffer(const DynamicRingBuffer &) noexcept = default;
      // move constructor
      constexpr DynamicRingBuffer(DynamicRingBuffer &&) noexcept = default;
      // copy assignment operator
      constexpr DynamicRingBuffer &operator=(const DynamicRingBuffer &) noexcept =
          default;
      // move assignment operator
      constexpr DynamicRingBuffer &operator=(DynamicRingBuffer &&) noexcept =
          default;
      // destructor
      ~DynamicRingBuffer() noexcept = default;

      bool push(T value) {
        if (full()) {
          return false;
        }

        _data[_tail] = value;
        _tail = (_tail + 1) % _data.size();
        _size++;

        return true;
      }

      constexpr std::optional<T> pop() noexcept {
        if (empty()) {
          return std::nullopt;
        }

        T value = _data[_head];
        _head = (_head + 1) % _data.size();
        _size--;

        return value;
      }

      constexpr size_t size() const { return _size; }

      constexpr size_t capacity() const { return S; }

      constexpr bool empty() const { return _size == 0; }

      constexpr bool full() const { return _size == S; }
    };
}  // namespace mr
```

#### Реализация №2 (динамический массив)
```cpp
#include <optional>
#include <vector>

namespace mr {
  template <typename T>
    struct DynamicRingBuffer {
    private:
      std::size_t _size;
      std::size_t _head;
      std::size_t _tail;
      std::vector<T> _data;

    public:
      // default constructor
      constexpr DynamicRingBuffer() noexcept = default;
      // copy constructor
      constexpr DynamicRingBuffer(const DynamicRingBuffer &) noexcept = default;
      // move constructor
      constexpr DynamicRingBuffer(DynamicRingBuffer &&) noexcept = default;
      // copy assignment operator
      constexpr DynamicRingBuffer &operator=(const DynamicRingBuffer &) noexcept =
          default;
      // move assignment operator
      constexpr DynamicRingBuffer &operator=(DynamicRingBuffer &&) noexcept =
          default;
      // destructor
      ~DynamicRingBuffer() noexcept = default;

      bool push(T value) {
        if (full()) {
          _data.resize(_size);
        }

        _data[_tail] = value;
        _tail = (_tail + 1) % _data.size();
        _size++;

        return true;
      }

      constexpr std::optional<T> pop() noexcept {
        if (empty()) {
          return std::nullopt;
        }

        T value = _data[_head];
        _head = (_head + 1) % _data.size();
        _size--;

        return value;
      }

      constexpr size_t size() const { return _size; }

      constexpr size_t capacity() const { return _data.capacity(); }

      constexpr bool empty() const { return _size == 0; }

      constexpr bool full() const { return _size == capacity(); }
    };
}  // namespace mr
```
- Реализация на статическом массиве более cache-friendly, так как не насаждает дополнительный уровень индирекции - данные находятся сразу в объекте
- При этом он меннее удобен в использовании, так как его размер нужно указать при компиляции. При этом оперировать объектами RingBuf разных размеров тоже будет не так удобно - нужно будет явно приводить типы.
- При изменении размера динамического RingBuf могут инвалидироваться указатели
- Так же стоит отметить, что динамическая реализация вероятно будет требовать mutex при изменении размера, тогда как статическая реализация известна простотой lock-free синхронизации.

### Вопрос 3
```cpp
#include <algorithm>
#include <iterator>

namespace mr {
  // standard quick sort's partition function
  template <typename It>
    It partition(It begin, It end) {
      using std::swap;
      swap(begin[std::distance(begin, end) / 2], end[-1]);
      auto pivot = end - 1;
      auto i = begin;
      for (auto j = begin; j != pivot; ++j) {
        if (*j < *pivot) {
          std::swap(*i, *j);
          i++;
        }
      }
      swap(*i, *pivot);
      return i;
    }

  template <typename It>
    void insertion_sort(It begin, It end) {
      auto distance = std::distance(begin, end);

      for (std::size_t i = 1; i < distance; i++) {
        auto tmp = begin[i];
        std::size_t j = i - 1;
        while (j >= 0 && tmp < begin[j]) {
          begin[j + 1] = begin[j];
          j--;
        }
        begin[j + 1] = tmp;
      }
    }

  // quick sort falling back to insertion sort
  template <typename It>
    void sort(It begin, It end) {
      // maximum number of elements fitting in a cacheline
      static constexpr int max_batch_size =
        std::hardware_destructive_interference_size /
        sizeof(std::remove_cvref_t<decltype(*begin)>);

      // divide until batch_size > max_batch_size
      if (std::distance(begin, end) >= max_batch_size) {
        // quick sort if not fitting in a cacheline
        auto mid = partition(begin, end);
        mr::sort(begin, mid);
        mr::sort(mid + 1, end);
      } else if (std::distance(begin, end) > 1) {
        // insertion_sort if fit in a cacheline
        insertion_sort(begin, end);
      }
    }
}
```
- Это реализация quick sort с insertion sort для малых подмассивов.
- Выбор опорного элемента в quick sort упрощен - выбирается центральный элемент. Использование "медианных" значений не показало ускорения в бенчмарках на случайных массивах.
- Размер, при котором алгоритм переключается на insertion sort равен количеству элементов массива, помещающихся в кеш-линию процессора. Размер кеш линии определяется встроенной переменной, но кроссплатформенное решение вероятно просто использует константу (как это делают реализации std::sort)
- Потенциальные оптимизации включают:
  - Специализацию для unsigned - сортировка подсчетом
  - Использование heapsort для минимизации времени на "неудачных" массивах - heapsort в случае "плохих" разделений
