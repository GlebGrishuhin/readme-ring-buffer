
В данном readme даны примеры работы с некоторыми сущностями библиотеки pde_solvers

## Оглавление
* [Организация хранения расчетных профилей в буферах pde_solvers](#Организация-хранения-расчетных-профилей-в-буферах-pde_solvers)
  * [Идея буфера pde_solvers](#Идея-буфера-pde_solvers)
  * [Пример использования буфера pde_solvers](#Пример-использования-буфера-pde_solvers)
* [Реализация метода Эйлера для решения дифференциальных уравнений](#Реализация-метода-Эйлера-для-решения-дифференциальных-уравнений)
  * [Идеи, лежащие в основе solve_euler](#Идеи,-лежащие-в-основе-solve_euler)
  * [Пример-использования-solve_euler ](#Пример-использования-solve_euler )


## Организация хранения расчетных профилей в буферах pde_solvers

Сущность `ring_buffer_t` предназначена для хранения расчетных профилей, содержащих
значения произвольного типа

### Идея буфера pde_solvers

При использовании метода характеристик для расчета профиля в текущий момент времени 
нужен профиль в предыдущий момент времени. На первый взгляд, кажется удачной идея 
организовать массив слоев, который будет дополняться все новыми  и новыми слоями 
в процессе расчета. Такой подход избыточен – в момент времени t<sub>k</sub>  для расчета слоя нужен лишь слой, соответствующий моменту t<sub>k-1</sub> 

Хорошей идеей выглядит создание массива размерности 2 для хранения текущего и предыдущего слоя. 
Но в процессе расчета текущий и предыдущий слои всё время меняются местами – как только 
рассчитан k-ый слой на основе (k-1)-ого, сам k-ый слой становится предыдущим для (k+1)-ого. 
Постоянно менять местами значения элементов массива не оптимально с точки зрения ресурсов. 
Решение – запоминать индекс текущего слоя, из предыдущих слоев хранить только необходимые для расчета.
Используется циклический сдвиг индекса текущего слоя. Положение самих слоев 
при этом не меняется (см. Рисунок).

![Схема, поясняющая идею циклического сдвига индекс текущего слоя в буфере][ring_buffer_schema]

[ring_buffer_schema]: images/ring_buffer_schema.png "Схема, поясняющая идею циклического сдвига индекс текущего слоя в буфере"  

<div align = "center">
</center><b>Схема, поясняющая идею циклического сдвига индекс текущего слоя в буфере</b></center>
</div>
<br>

Чтобы еще больше упросить себе жизнь, нужно запоминать индекс текущего слоя в самом буфере. 
Решение – реализовать логику смещения индекса внутри самого буфера. Вырисовывается реализация 
буфера как класса. Слои и текущий индекс будут данными класса. Методы должны позволять менять 
слой, который считается текущим; получать текущий слой и предыдущий.
Описанные идеи реализованы в буфере pde_solvers. Методы буфера pde_solvers освобождают от 
необходимости запоминать индекс текущего слоя вне самого буфера.

Метод ```.advance(i)```
циклически сдвигает индекс текущего слоя на заданную величину *i*.  

Метод ```.current()```
возвращает ссылку на текущий слой буфера.  

Метод ```.previous()```
возвращает ссылку на предыдущий слой буфера.

### Пример использования буфера pde_solvers

Предположим, что перед нами стоит задача расчета движения партий. Будем моделировать 
изменение плотности жидкости в трубе при помощи метода характеристик. 

**Решение задачи многослойного расчета плотности методом характеристик с сохранением текущего и предыдущего слоя в буфере выглядит следующим образом:**

```C++
// Длина трубы, м
double L = 3000;
// Количество точек расчетной сетки
int grid_size = 100;
// Граничное условие для плотности
double left_condition_density = 860;
// Начальное значение плотности
double initial_condition_density = 850;
// Cкорость потока, м/с
double v = 1.5;
// Величина шага между узлами расчетной сетки, м
double dx = L / (grid_size - 1);
// Шаг во времени из условия Куранта
double dt = dx / v;

// Время моделирования, с
// Вместе со значением числа Куранта определяет, 
// какое количество слоев будет рассчитано
double T = 10e3; // время моделирования участка трубопровода, с 

// Создаем слой, заполненный начальным значением плотности
// Далее используем его для инициализации слоев буфера
vector<double> initial_layer(grid_size, initial_condition_density);

// Создание буфера для решаемой задачи
// 2 - количество слоев в буфере
// initial_layer - слой, значениями из которого проинициализируются все слои буфера
ring_buffer_t<vector<double>> buffer(2, initial_layer);

for (double time = 0; time < T; time += dt)
{
	// Получение ссылок на текущий и предыдущий слои буфера
	vector<double>& previous_layer = buffer.previous();
	vector<double>& current_layer = buffer.current();

	// Расчет методом характеристик
	// Суть - смещение предыдущего слоя и запись граничного условия
	for (size_t i = 1; i < current_layer.size(); i++)
	{
		current_layer[i] = previous_layer[i - 1];
	}
	current_layer[0] = left_condition_density;
	
	// Слой current_layer на следующем шаге должен стать предыдущим. 
	// Для этого сместим индекс текущего слоя в буфере на единицу
	buffer.advance(+1);
}
```

## Реализация метода Эйлера для решения дифференциальных уравнений

### Идеи, лежащие в основе solve_euler

`solve_euler` это функция, которая решает математическую задачу – находит решение заданной системы 
обыкновенных дифференциальных уравнений (ОДУ) методом Эйлера. Функция ничего не знает о трубе и 
процессах в ней. `solve_euler` работает лишь с системой ОДУ, заданной на расчётной сетке.

В `pde_solvers` система ОДУ представлена базовым классом `ode_t`. Система уравнений 
в частных производных представлена базовым классом `pde_t`. 

На уровне идеи эти сущности представляют собой матричную запись системы дифференциальных уравнений. 
Эти сущности предоставляют методы, позволяющие численно найти решение ДУ.

Численное решение ОДУ методом Эйлера предполагает:
* расчет производной f'(x<sub>i</sub> ) в i-ом узле расчетной сетки
* оценку приращения функции по значению производной
 ∆y = y<sub>i</sub> +f'(x<sub>i</sub> )∙(x<sub>i+1</sub>-x<sub>i</sub>) = f(x<sub>i</sub> )+f'(x<sub>i</sub> )∙(x<sub>i+1</sub> -x<sub>i</sub>)
* расчет значения функции в следующем узле сетки на основании оценки приращения функции: y<sub>i+1</sub>=y<sub>i</sub>+∆y

Соответственно, методы `ode_t` должны позволять рассчитывать значение производной 
в данной точке расчетной сетки или при данных значениях аргументов.

Методы базовых классов ode_t имеют одинаковый набор параметров:
`имя_метода(grid_index, point_vector)`
* где `grid_index` - индекс текущего узла расчетной сетки. Его можно использовать 
для дифференцирования параметра, профиль которого на данной расчетной сетке известен 
(например, гидростатический перепад dz/dt при известном профиле трубы);
* `point_vector` – вектор значений неизвестных в данной точке.



`solve_euler` работает с сущностями типа ode_t и ничего не знает о физическом смысле 
решаемой задачи. Сведение определенной задачи трубопроводного транспорта 
к базовому классу ode_t или pde_t – ответственность разработчика, использующего `pde_solvers`. 
Это сведение выполняется созданием класса, производного от базовых классов `ode_t или pde_t`. 
Производные классы переопределяют поведение функций базового класса  на основе математической модели задачи.


### Пример использования solve_euler

Используем solve_euler для расчет профиля давления в классической задаче PQ 
(см. Лурье 2012, раздел 4.2, Задаче 1)

** Решаемое диффеенциальное уравнение реализовано следующим образом:**
```C++
/// @brief Уравнение трубы для задачи PQ
class Pipe_model_for_PQ_t : public ode_t<1>
{
public:
    using ode_t<1>::equation_coeffs_type;
    using ode_t<1>::right_party_type;
    using ode_t<1>::var_type;
protected:
    pipe_properties_t& pipe;
    oil_parameters_t& oil;
    double flow;

public:
    /// @brief Констуктор уравнения трубы
    /// @param pipe Ссылка на сущность трубы
    /// @param oil Ссылка на сущность нефти
    /// @param flow Объемный расход
    Pipe_model_for_PQ_t(pipe_properties_t& pipe, oil_parameters_t& oil, double flow)
        : pipe(pipe)
        , oil(oil)
        , flow(flow)
    {

    }

    /// @brief Возвращает известную уравнению сетку
    virtual const vector<double>& get_grid() const override {
        return pipe.profile.coordinates;
    }

    /// @brief Возвращает значение правой части ДУ
    /// см. файл 2023-11-09 Реализация стационарных моделей с прицелом на квазистационар.docx
    /// @param grid_index Обсчитываемый индекс расчетной сетки
    /// @param point_vector Начальные условия
    /// @return Значение правой части ДУ в точке point_vector
    virtual right_party_type ode_right_party(
        size_t grid_index, const var_type& point_vector) const override
    {
        double rho = oil.density();
        double S_0 = pipe.wall.getArea();
        double v = flow / (S_0);
        double Re = v * pipe.wall.diameter / oil.viscosity();
        double lambda = pipe.resistance_function(Re, pipe.wall.relativeRoughness());
        double tau_w = lambda / 8 * rho * v * abs(v);
        /// Обработка индекса в случае расчетов на границах трубы
        /// Чтобы не выйти за массив высот, будем считать dz/dx в соседней точке
        grid_index = grid_index == 0 ? grid_index + 1 : grid_index;
        grid_index = grid_index == pipe.profile.heights.size() - 1 ? grid_index - 1 : grid_index;

        double height_derivative = (pipe.profile.heights[grid_index] - pipe.profile.heights[grid_index - 1]) /
            (pipe.profile.coordinates[grid_index] - pipe.profile.coordinates[grid_index - 1]);

        return { ((-4) / pipe.wall.diameter) * tau_w - rho * M_G * height_derivative };
    }

};
```
** Применение solve_euler к разработаному классу `Pipe_model_for_PQ_t`  **

```C++
int main()
{
    /// Создаем сущность трубы
    pipe_properties_t pipe;

    /// Задаем сетку трубы
    pipe.profile.coordinates = { 0, 1000, 2000 };

    /// Задаем высотные отметки и 
    pipe.profile.heights = vector<double>(pipe.profile.coordinates.size(), 0);

    /// Создаем буфер из двух слоев, каждый совпадает по размеру с pipe.profile.heights
    ring_buffer_t<vector<double>> buffer(2, pipe.profile.heights);

    /// Создаем сущность нефти
    oil_parameters_t oil;

    /// Задаем объемнй расход нефти, [м3/с]
    double Q = 0.8;

    /// Создаем расчетную модель трубы
    Pipe_model_for_PQ_t pipeModel(pipe, oil, Q);

    /// Получаем указатель на начало слоя в буфере
    profile_wrapper<double, 1> start_layer(buffer.current());

    ///Задаем начальное давление
    double Pout = 5e5;

    /// Модифицированный метод Эйлера для модели pipeModel,
    /// расчет ведется справа-налево относительно сетки,
    /// начальное условие Pout, 
    /// результаты расчета запишутся в слой, на который указывает start_layer
    solve_euler_corrector<1>(pipeModel, -1,  Pout , &start_layer);
}
```