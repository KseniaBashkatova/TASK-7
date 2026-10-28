## Практическая работа № 7
```Csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace SmartGridControl
{
    class Program
    {
        // Константы сети
        private const double TARGET_FREQUENCY = 50.0;
        private const double CASCADE_DECAY_THRESHOLD = 0.1;

        static void Main(string[] args)
        {
            Console.WriteLine("=== Центр управления энергосетью (Smart Grid) ===\n");

            // 1. Блок Студента 1
            SystemCoreEngineer();

            // 2. Блок Студента 2
            AnalyticsEngineer();

            Console.WriteLine("\n=== Симуляция завершена ===");
        }

        static void SystemCoreEngineer()
        {
            // Трансферная загрузка (ref + out)
            int sourceLoad = 850;
            int targetCapacity = 400;
            int requestedMw = 300;

            Console.WriteLine("--- Балансировка нагрузки (TransferLoad) ---");
            Console.WriteLine($"Исходно: Источник={sourceLoad} МВт, Приемник={targetCapacity} МВт, Запрос={requestedMw} МВт");
            TransferLoad(ref sourceLoad, ref targetCapacity, requestedMw, out bool isOverloaded, out int lossMegawatts);
            Console.WriteLine($"Результат: Источник={sourceLoad} МВт, Приемник={targetCapacity} МВт");
            Console.WriteLine($"Перегрузка линии: {isOverloaded}, Технические потери: {lossMegawatts} МВт\n");

            // StabilizeFrequency (перегрузка)
            double gridHz = 49.7;
            Console.WriteLine("--- Стабилизация частоты (StabilizeFrequency) ---");
            Console.WriteLine($"Текущая частота: {gridHz} Гц");
            StabilizeFrequency(ref gridHz);
            Console.WriteLine($"После базовой подстройки: {gridHz} Гц\n");

            double hydroReserve = 150.0;
            double thermalReserve = 300.0;
            double deviation;
            StabilizeFrequency(ref gridHz, hydroReserve, thermalReserve, out deviation);
            Console.WriteLine($"После подключения резервов (ГЭС={hydroReserve} МВт, ТЭС={thermalReserve} МВт): {gridHz} Гц");
            Console.WriteLine($"Остаточное отклонение: {deviation} Гц\n");

            // CascadeFailureProbability (рекурсия)
            Console.WriteLine("--- Расчет каскадного отключения (Рекурсия) ---");
            int nodes = 7;
            double initialStress = 0.9;
            double probability = CalculateCascade(nodes, initialStress, 0);
            Console.WriteLine($"Вероятность каскадного отключения для {nodes} узлов: {probability:P2}\n");
        }

        // переброска мощности с расчетом потерь и перегрузки
        static void TransferLoad(ref int sourceLoad, ref int targetCapacity, int requestedMw, out bool isOverloaded, out int lossMegawatts)
        {
            // Расчет потерь на сопротивлении (упрощенная модель: 2% от передаваемой мощности)
            lossMegawatts = (int)(requestedMw * 0.02);
            int actualTransfer = requestedMw - lossMegawatts;

            sourceLoad -= actualTransfer;
            targetCapacity += actualTransfer;

            // Проверка перегрузки (если нагрузка источника упала ниже 10% или емкость приемника выше 95%)
            isOverloaded = (sourceLoad < (sourceLoad + actualTransfer) * 0.1) || targetCapacity > 950;
        }

        // Перегрузка 1 
        static void StabilizeFrequency(ref double gridHz)
        {
            double correction = (TARGET_FREQUENCY - gridHz) * 0.5;
            gridHz += correction;
            if (Math.Abs(gridHz - TARGET_FREQUENCY) < 0.01) gridHz = TARGET_FREQUENCY;
        }

        // Перегрузка 2 
        static void StabilizeFrequency(ref double gridHz, double hydroReserveMw, double thermalReserveMw, out double deviation)
        {
            // каждые 100 МВт резерва дают +0.1 Гц к частоте при просадке
            double totalReserve = hydroReserveMw + thermalReserveMw;
            double boost = (totalReserve / 100.0) * 0.1;

            if (gridHz < TARGET_FREQUENCY)
            {
                gridHz += boost;
                if (gridHz > TARGET_FREQUENCY) gridHz = TARGET_FREQUENCY;
            }

            deviation = Math.Abs(TARGET_FREQUENCY - gridHz);
        }

        // рекурсивный расчет цепной реакции
        static double CalculateCascade(int nodeCount, double currentStress, int step)
        {
            // Базовый случай: затухание напряжения или конец цепи
            if (currentStress < CASCADE_DECAY_THRESHOLD || step >= nodeCount)
            {
                return 0.0;
            }

            // Вероятность отказа текущего узла на данном шаге
            double currentFailureProb = currentStress * 0.4;

            // Напряжение падает на каждом шаге
            double nextStress = currentStress * 0.7;

            // Рекурсивный вызов для следующего узла
            double downstreamProb = CalculateCascade(nodeCount, nextStress, step + 1);

            // Общая вероятность: отказ текущего + вероятность того что он вызвал отказ следующего
            return currentFailureProb + (currentFailureProb * downstreamProb);
        }

        static void AnalyticsEngineer()
        {
            // SumDistrictDeficit (параметры)
            int[] districtDemands = { 120, 450, 80, 610, 300, 95 };
            int thresholdLimit = 500;

            Console.WriteLine("--- Анализ дефицита районов (Params) ---");
            Console.WriteLine($"Заявки районов: {string.Join(", ", districtDemands)}");
            Console.WriteLine($"Критический предел: {thresholdLimit} МВт");
            int totalDeficit = AnalyzeDeficit(thresholdLimit, districtDemands);
            Console.WriteLine($"Суммарный дефицит по районам сверх лимита: {totalDeficit} МВт\n");

            // (матрица + анализ)
            int[,] powerMatrix = {
                { 100, 50, 20, 0 },
                { 50, 200, 80, 30 },
                { 20, 80, 150, 60 },
                { 0, 30, 60, 90 }
            };

            Console.WriteLine("--- Анализ топологии сети (Матрица) ---");
            RenderGridTopology(powerMatrix);
        }

        // агрегация с параметрами
        static int AnalyzeDeficit(int thresholdLimit, params int[] districtDemands)
        {
            if (districtDemands == null || districtDemands.Length == 0) return 0;

            int deficitSum = 0;
            foreach (int demand in districtDemands)
            {
                if (demand > thresholdLimit)
                {
                    // Дефицит считается как превышение над порогом
                    deficitSum += (demand - thresholdLimit);
                }
            }
            return deficitSum;
        }

        // анализ матрицы связности
        static void RenderGridTopology(int[,] powerMatrix)
        {
            int sectors = powerMatrix.GetLength(0);
            int[] sectorSums = new int[sectors];
            int maxRiskValue = -1;
            (int, int) maxRiskNode = (-1, -1);

            Console.WriteLine("Тепловая карта нагрузок (МВт):");

            for (int i = 0; i < sectors; i++)
            {
                int rowSum = 0;
                for (int j = 0; j < powerMatrix.GetLength(1); j++)
                {
                    rowSum += powerMatrix[i, j];

                    // Поиск максимального риска
                    if (i == j && powerMatrix[i, j] > maxRiskValue)
                    {
                        maxRiskValue = powerMatrix[i, j];
                        maxRiskNode = (i, j);
                    }

                    // Визуализация тепловой карты (нагрузка > 100 - критическая)
                    if (powerMatrix[i, j] > 100) Console.Write("[HHH] ");
                    else if (powerMatrix[i, j] > 50) Console.Write("[MMM] ");
                    else if (powerMatrix[i, j] > 0) Console.Write("[LLL] ");
                    else Console.Write("[---] ");
                }
                sectorSums[i] = rowSum;
                Console.WriteLine();
            }

            Console.WriteLine("\nСуммарная передаваемая мощность по секторам (строкам):");
            for (int i = 0; i < sectorSums.Length; i++)
            {
                Console.WriteLine($"Сектор {i + 1}: {sectorSums[i]} МВт");
            }

            Console.WriteLine($"\nУзловая точка максимального риска короткого замыкания: " +
                              $"Узел ({maxRiskNode.Item1 + 1}, {maxRiskNode.Item2 + 1}) " +
                              $"с нагрузкой {maxRiskValue} МВт");
        }
    }
}
```
!Результат выполнения:
<img width="1437" height="853" alt="scr 1" src="https://github.com/user-attachments/assets/79a828e9-653b-43cf-ba0e-d4adc7bf2699" />
<img width="1430" height="825" alt="scr 2" src="https://github.com/user-attachments/assets/c6589c73-0027-41d0-83b2-cb11e291a289" />
<img width="1452" height="828" alt="scr 3" src="https://github.com/user-attachments/assets/4e171131-4e30-495c-aaf6-fbe1360ffab4" />
<img width="1810" height="811" alt="scr 4" src="https://github.com/user-attachments/assets/67fd07c2-a9f4-49b0-9e7d-dc9fa9abd614" />
// Делали Аристов и Башкатова
