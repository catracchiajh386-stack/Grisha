from datetime import datetime, timedelta
import json
import random
import time

# --- 🔌 Класс Зарядной Сессии (Charging Session) ---
class ChargingSession:
    """Представляет одну активную или завершенную сессию зарядки."""
    def __init__(self, connector_id, user_id):
        self.session_id = random.randint(1000, 9999)
        self.connector_id = connector_id
        self.user_id = user_id
        self.start_time = datetime.now()
        self.end_time = None
        self.start_meter = 0.0 # Показания счетчика в начале (kWh)
        self.end_meter = 0.0   # Показания счетчика в конце (kWh)
        self.status = "ACTIVE"
        print(f"-> Сессия {self.session_id} начата на коннекторе {connector_id} для пользователя {user_id}.")

    def stop(self, end_meter_reading):
        """Завершает сессию зарядки."""
        self.end_meter = end_meter_reading
        self.end_time = datetime.now()
        self.status = "FINISHED"
        print(f"<- Сессия {self.session_id} завершена. Конечное показание: {self.end_meter:.2f} kWh.")

    def get_energy_consumed(self):
        """Рассчитывает потребленную энергию (kWh)."""
        return self.end_meter - self.start_meter

    def to_dict(self):
        """Возвращает данные сессии для логирования и расчетов."""
        return {
            "session_id": self.session_id,
            "connector_id": self.connector_id,
            "user_id": self.user_id,
            "status": self.status,
            "start_time": self.start_time.isoformat(),
            "end_time": self.end_time.isoformat() if self.end_time else None,
            "energy_consumed_kWh": self.get_energy_consumed(),
        }

# --- ⚡️ Класс Зарядной Станции (EV Charging Station) ---
class EVChargingStation:
    """Центральный контроллер, управляющий тарифами и сессиями."""
    
    # Тарифы (примеры реальных тарифов)
    TARIFF = {
        "energy_rate": 8.50,    # Цена за 1 kWh (руб.)
        "time_rate_per_min": 1.0, # Цена за простой (руб./мин)
        "free_time_minutes": 5  # Бесплатное время простоя после окончания зарядки
    }
    
    def __init__(self, station_id, num_connectors=2):
        self.station_id = station_id
        self.connectors = {i: "AVAILABLE" for i in range(1, num_connectors + 1)}
        self.active_sessions = {}  # {connector_id: ChargingSession object}
        self.history = []

    def start_charging(self, connector_id, user_id, start_meter):
        """Инициирует новую зарядную сессию."""
        if self.connectors.get(connector_id) != "AVAILABLE":
            print(f"❌ Коннектор {connector_id} занят.")
            return False
        
        session = ChargingSession(connector_id, user_id)
        session.start_meter = start_meter # Устанавливаем начальное показание
        
        self.connectors[connector_id] = "CHARGING"
        self.active_sessions[connector_id] = session
        return True

    def stop_charging(self, connector_id, end_meter):
        """Завершает сессию и рассчитывает стоимость."""
        if connector_id not in self.active_sessions:
            print(f"❌ На коннекторе {connector_id} нет активной сессии.")
            return None
        
        session = self.active_sessions[connector_id]
        session.stop(end_meter)
        
        cost_details = self._calculate_cost(session)
        
        # Обновление состояния станции
        self.history.append(session.to_dict())
        del self.active_sessions[connector_id]
        self.connectors[connector_id] = "AVAILABLE"
        
        return cost_details

    def _calculate_cost(self, session):
        """Рассчитывает стоимость сессии на основе энергии и времени."""
        energy_consumed = session.get_energy_consumed()
        
        # Расчет по энергии
        energy_cost = energy_consumed * self.TARIFF["energy_rate"]
        
        # Расчет по времени простоя (для упрощения примем, что вся сессия была зарядкой)
        # В реальной системе здесь нужно отличать время ЗАРЯДКИ от времени ПРОСТОЯ (Idle time)
        
        total_duration = (session.end_time - session.start_time).total_seconds() / 60
        
        # Имитация начисления за простой (если сессия была бы очень длинной)
        idle_duration = max(0, total_duration - 60) # Условно, бесплатный час зарядки
        idle_cost = idle_duration * self.TARIFF["time_rate_per_min"]

        total_cost = energy_cost + idle_cost
        
        return {
            "session_id": session.session_id,
            "energy_consumed_kWh": f"{energy_consumed:.2f}",
            "energy_cost": f"{energy_cost:.2f} руб.",
            "idle_duration_min": f"{idle_duration:.1f}",
            "idle_cost": f"{idle_cost:.2f} руб.",
            "TOTAL_DUE": f"{total_cost:.2f} руб."
        }

    def display_status(self):
        """Выводит текущий статус коннекторов."""
        print(f"\n--- Статус станции {self.station_id} ---")
        for cid, status in self.connectors.items():
            session_info = f" (Sess ID: {self.active_sessions[cid].session_id})" if cid in self.active_sessions else ""
            print(f"Коннектор {cid}: {status}{session_info}")
        print("-" * 30)
