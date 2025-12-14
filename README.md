import pandas as pd
from datetime import datetime
import os

LOG_FILE = 'sales_call_log.csv'

def analyze_call_performance(log_file):
    """
    Загружает журнал звонков и рассчитывает ключевые метрики эффективности.
    """
    if not os.path.exists(log_file):
        print(f"❌ Файл журнала '{log_file}' не найден. Невозможно провести анализ.")
        return

    try:
        df = pd.read_csv(log_file)
        
        # Преобразование метки времени
        df['timestamp'] = pd.to_datetime(df['timestamp'])
        df['hour'] = df['timestamp'].dt.hour
        df['date'] = df['timestamp'].dt.date
        
        total_calls = len(df)
        
        if total_calls == 0:
            print("Журнал пуст. Нет данных для анализа.")
            return

        print(f"\n--- АНАЛИТИЧЕСКИЙ ОТЧЕТ: Продажи по телефону ({total_calls} звонков) ---")
        
        # --- Метрика 1: Общая конверсия ---
        
        # Определяем "Успешный" исход как назначение встречи или прямую продажу
        successful_outcomes = ['Назначена встреча', 'Продажа']
        successful_calls = df[df['outcome'].isin(successful_outcomes)]
        
        total_successful = len(successful_calls)
        conversion_rate = (total_successful / total_calls) * 100
        
        print(f"\n1. Конверсия:")
        print(f"   Общее количество успешных контактов: {total_successful}")
        print(f"   Уровень конверсии (успех/всего): {conversion_rate:.2f}%")
        
        # --- Метрика 2: Эффективность по исходу ---
        print("\n2. Распределение исходов звонков:")
        outcome_counts = df['outcome'].value_counts(normalize=True) * 100
        print(outcome_counts.to_string(float_format="%.1f%%"))
        
        # --- Метрика 3: Оптимальное время звонка (по конверсии) ---
        print("\n3. Лучшее время для звонка (по часам):")
        
        # Группировка по часу звонка и расчет конверсии
        hourly_data = df.groupby('hour').agg(
            total_calls=('outcome', 'count'),
            successful_calls=('outcome', lambda x: x.isin(successful_outcomes).sum())
        )
        hourly_data['conversion_rate'] = (hourly_data['successful_calls'] / hourly_data['total_calls']) * 100
        
        # Сортировка и вывод 3 лучших часов
        top_hours = hourly_data.sort_values(by='conversion_rate', ascending=False).head(3)
        
        for index, row in top_hours.iterrows():
            print(f"   Час {index:02d}:00 (Всего: {row['total_calls']} звонков) -> Конверсия: {row['conversion_rate']:.2f}%")
            
        # --- Метрика 4: Средняя длительность звонка ---
        avg_duration = df['duration_seconds'].mean()
        print(f"\n4. Средняя длительность всех звонков: {avg_duration:.1f} секунд")
        
        # --- Метрика 5: Анализ эффективности операторов (если есть столбец 'operator') ---
        if 'operator' in df.columns:
            operator_summary = df.groupby('operator').agg(
                total_calls=('outcome', 'count'),
                successful_calls=('outcome', lambda x: x.isin(successful_outcomes).sum()),
                avg_duration=('duration_seconds', 'mean')
            )
            operator_summary['conversion_rate'] = (operator_summary['successful_calls'] / operator_summary['total_calls']) * 1
