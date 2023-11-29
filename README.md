# Организация хранения расчетных профилей в буферах pde_solvers

Сущность `ring_buffer_t` предназначена для хранения расчетых профилей, содержащих
значения произвольного типа

## Оглавление
* [Идея буфера pde_solvers](#Идея-буфера-pde_solvers)
* [Пример использования буфера pde_solvers](#Пример-использования-буфера-pde_solvers)

## Идея буфера pde_solvers

При использовании метода характеристик для расчета профиля в текущий момент времени 
нужен профиль в предыдущий момент времени. На первый взгляд, кажется удачной идея 
организовать массив слоев, который будет дополняться все новыми  и новыми слоями 
в процессе расчета. Такой подход избыточен – в момент времени t<sub>k</sub>  для расчета слоя нужен лишь слой, соответствующий моменту t<sub>k-1</sub> 

Хорошей идеей выглядит создание массива размерности 2 для хранения текущего и предыдущего слоя. 
Но в процессе расчета текущий и предыдущий слои всё время меняются местами – как только 
рассчитан k-ый слой на основе (k-1)-ого, сам k-ый слой становится предыдущим для (k+1)-ого. 
Постоянно менять местами значения элементов массива не оптимально с точки зрения ресурсов. 
Решение – запоминать индекс текущего слоя. Чтобы уместить большое число слоев в буфер малой 
размерности, используется циклический сдвиг индекса текущего слоя. Положение самих слоев 
при этом не меняется (см. Рисунок).

![Схема, поясняющая идею циклического сдвига индекс текущего слоя в буфере][ring_buffer_schema]

[ring_buffer_schema]: images/ring_buffer_schema.png "Схема, поясняющая идею циклического сдвига индекс текущего слоя в буфере"

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

## Пример использования буфера pde_solvers

Предположим, что перед нами стоит задача расчета движения партий. Будем моделировать 
изменение плотности жидкости в трубе при помощи метода характеристик. Буфер pde_solvers 
отлично подходит для работы с расчетными слоями.

Буфер предназначен для хранения контейнеров  - наборов объектов определенного типа в C++. 
Для создания буфера необходимо указать тип слоев, которые будут храниться в нем. 
В простейшем случае можно использовать `vector<double>`. 
В pde_solvers реализован тип `profile_collection_t` для работы с профилями параметров, но его возможности пока что избыточны – освоить работу буферов можно без него. 

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