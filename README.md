# Data_analysis_mobile_game

Представим, я работаю в компании, которая разрабатывает мобильные игры. Ко мне пришел менеджер с рядом задач по исследованию нескольких аспектов мобильного приложения.

В первую очередь, его интересует показатель Retention. Мне нужно написать функцию для его подсчета. Помимо этого, в компании провели A/B тестирование наборов акционных предложений. На основе имеющихся данных мне нужно определить, какой набор можно считать лучшим и на основе каких метрик стоит принять правильное решение. И предложить метрики для оценки результатов последнего прошедшего тематического события в игре.

# Полученные результаты

1. Проведен анализ на Python и подготовлена функция для расчета метрики Retention Rate (RR) пользователей по дням со дня регистрации (в %) и для визуализации построена тепловая карта. Функция позволяет выбирать нужный временной интервал для даты регистрации. Рекомендуется отображать на тепловой карте не более 1 месяца. На тепловой карте отображается краткосрочный RR на протяжении 30 дней со дня регистрации. При необходимости, можно рассмотреть и долгосрочный RR на протяжении года.

<p align="center">

  <img width="760" height="500" src="https://github.com/Juldid/Data_analysis_mobile_game/blob/main/Cohorts.png">

</p>

```python
def RR_by_days(path_to_file_reg, path_to_file_auth, min_date, max_date):
    """
    Считает метрику Retention Rate пользователей по дням со дня регистрации (в %) и строит тепловую карту.
            Параметры:
                    path_to_file_reg (csv): путь к файлу о времени регистрации пользователей. 
                    Датафрейм имеет структуру:
                        reg_ts (timestamp) - время регистрации пользователя
                        uid (int) - ID пользователя
                        
                    path_to_file_auth (csv): путь к файлу о времени захода пользователей в игру (авторизации). 
                    Датафрейм имеет структуру:
                        auth_ts (timestamp) - время авторизации пользователя
                        uid (int) - ID пользователя
                    
                    min_date (str):  min дата диапазона в формате "Y-m-d" 
                    max_date (str): max дата диапазона в формате "Y-m-d"
                    
            Возвращаемое значение:
                    heatmap - тепловая карта с метрикой Retention Rate пользователей (в %) по дням со дня регистрации
    """

    reg_data = pd.read_csv(filepath_or_buffer=path_to_file_reg, sep=';') 
    # Считываем файл с временем регистрации пользователей
    auth_data = pd.read_csv(filepath_or_buffer=path_to_file_auth, sep=';')
    # Считываем файл с временем авторизации пользователей

    reg_data['reg_ts'] = pd.to_datetime(reg_data['reg_ts'], unit='s')  # Конвертируем  даты из timestamp в datetime 
    auth_data['auth_ts'] = pd.to_datetime(auth_data['auth_ts'], unit='s')
    
    max_date = datetime.strptime(max_date, '%Y-%m-%d')
    max_date_auth = max_date + timedelta(days=30) 
    # Добавляем к max_date 30 дней (т.к. когортный анализ делаем на 30 дней со дня регистрации)
    
    reg_data = reg_data.query('@min_date <= reg_ts <= @max_date')  # Выбираем нужный период
    auth_data = auth_data.query('@min_date <= auth_ts <= @max_date_auth')

    
    data = reg_data.merge(auth_data, how='inner', on='uid')
    
    data['reg_ts'] = data.reg_ts.dt.strftime('%Y-%m-%d')  # Скорректируем формат даты 
    data['auth_ts'] = data.auth_ts.dt.strftime('%Y-%m-%d')
    data['reg_ts'] = pd.to_datetime(data['reg_ts'])
    data['auth_ts'] = pd.to_datetime(data['auth_ts'])
    
    data['diff'] = data.auth_ts.dt.to_period('D').astype(int) - data.reg_ts.dt.to_period('D').astype(int) 
    # Посчитаем разницу между датой регистрации и датой авторизации (в днях)
 
    reg_users = data.groupby('reg_ts')\
        .agg({'uid': 'nunique'})\
        .reset_index()\
        .rename(columns={'uid': 'count_reg_users'}) 
    # Считаем кол-во уникальных зарегистрированных пользователей (по дням)
    
    auth_users = data.query('diff != 0')\
        .groupby(['reg_ts', 'diff'])\
        .agg({'uid': 'nunique'})\
        .reset_index()\
        .rename(columns={'uid': 'count_auth_users'})
    # Считаем кол-во уникальных авторизованных (зашедших в игру) пользователей (по дням)
    
    data_result = reg_users.merge(auth_users, how='left', on='reg_ts')
    # Объединяем датафреймы
    
    data_result['RR'] = (100 * (data_result['count_auth_users'] / data_result['count_reg_users'])).round(2)
    # Посчитаем Retention Rate (в %) по дням
    
    data_result['reg_ts'] = data_result.reg_ts.astype(str)
    
    data_result_month = data_result.query('diff <= 30')
    # Смотрим только первые 30 дней со дня регистрации
    
    data_result_pivot = data_result_month.pivot_table(index='reg_ts', columns='diff', values='RR').fillna(0) 
    # Построим pivot-таблицу
    
    plt.subplots(figsize=(20, 10))
    sns.heatmap(data_result_pivot, annot=True, fmt="g").set(xlabel="Day", ylabel="Reg_date")
    plt.show()
    # Построим тепловую карту
```

2. Работа функции проверена на временном интервале с 01.06.2019 по 30.06.2019. Мы наблюдаем низкий Retention первого дня (D1) < 2% авторизаций пользователей. Возможно это связано с процессом регистрации, авторизации или оплаты пользователями. Потом наблюдаем увеличение RR на протяжении первых 10-14 дней (наше среднее время удержания в игре) со дня регистрации до 8% и плавный спад к концу месяца. Рекомендуется провести анализ причин низкого RR в первые дни после регистрации.

3. По результатам проведенного А/В тестирования проанализированы метрики конверсии в покупку (CR), средний доход на одного пользователя (ARPU) и средний доход на одного платящего пользователя (ARPPU), написана функция проверки гипотез методом bootstrap и получено, что стат значимых раличий анализируемых метрик в контрольной и тестовой группах нет. Поэтому дать однозначный ответ, какое акционное предложение является лучшим с точки зрения финансовых показателей, мы не можем. 

```python
def get_bootstrap( 
    data,  # датафрейм с данными по группам 
    boot_it,  # количество бутстрэп-подвыборок 
    bootstrap_conf_level,  # уровень значимости
    metrica=''  # анализируемая метрика, default metrica = CR
):
    boot_data = []
    boot_len = max([len(data.query("testgroup == 'a'")), len(data.query("testgroup == 'b'"))])
    for i in tqdm(range(boot_it)):  # извлекаем подвыборки
        samples = data.sample(boot_len, replace=True)
        if metrica == 'CR' or metrica == '':
            CR_1 = samples.query("testgroup == 'a'").agg({'user_id': 'count'}).user_id\
                / samples.query("testgroup == 'a' and revenue != 0").agg({'user_id': 'count'}).user_id
            CR_2 = samples.query("testgroup == 'b'").agg({'user_id': 'count'}).user_id\
                / samples.query("testgroup == 'b' and revenue != 0").agg({'user_id': 'count'}).user_id
            boot_data.append(CR_1 - CR_2)  # CR
        elif metrica == 'ARPU':
            ARPU_1 = samples.query("testgroup == 'a'").agg({'revenue': 'sum'}).revenue\
                / samples.query("testgroup == 'a'").agg({'user_id': 'count'}).user_id
            ARPU_2 = samples.query("testgroup == 'b'").agg({'revenue': 'sum'}).revenue\
                / samples.query("testgroup == 'b'").agg({'user_id': 'count'}).user_id
            boot_data.append(ARPU_1 - ARPU_2)  # ARPU
        elif metrica == 'ARPPU':
            ARPPU_1 = samples.query("testgroup == 'a' and revenue != 0").agg({'revenue': 'sum'}).revenue\
                / samples.query("testgroup == 'a' and revenue != 0").agg({'user_id': 'count'}).user_id
            ARPPU_2 = samples.query("testgroup == 'b' and revenue != 0").agg({'revenue': 'sum'}).revenue\
                / samples.query("testgroup == 'b' and revenue != 0").agg({'user_id': 'count'}).user_id
            boot_data.append(ARPPU_1 - ARPPU_2)  # ARPPU
        
    pd_boot_data = pd.DataFrame(boot_data)
        
    left_quant = (1 - bootstrap_conf_level) / 2

    right_quant = 1 - (1 - bootstrap_conf_level) / 2
    quants = pd_boot_data.quantile([left_quant, right_quant])
        
    p_1 = norm.cdf(
        x=0, 
        loc=np.mean(boot_data), 
        scale=np.std(boot_data)
    )
    p_2 = norm.cdf(
        x=0, 
        loc=-np.mean(boot_data), 
        scale=np.std(boot_data)
    )
    p_value = min(p_1, p_2) * 2
    
    # Визуализация
    _, _, bars = plt.hist(pd_boot_data[0], bins=50)
    for bar in bars:
        if bar.get_x() <= quants.iloc[0][0] or bar.get_x() >= quants.iloc[1][0]:
            bar.set_facecolor('red')
        else: 
            bar.set_facecolor('grey')
            bar.set_edgecolor('black')
    
    plt.style.use('ggplot')
    plt.vlines(quants, ymin=0, ymax=50, linestyle='--')
    plt.xlabel('boot_data')
    plt.ylabel('frequency')
    plt.title("Histogram of boot_data")
    plt.show()
       
    return {"boot_data": boot_data, 
            "quants": quants, 
            "p_value": p_value}
```

4. Также мы посмотрели на распределение дохода (revenue) в группах и получили следующие результаты. В контрольной группе мы имеем большую часть покупателей с низкодоходными покупками (revenue < 500) и ряд покупателей с высокодоходными покупкам (revenue > 5000), которые следует отдельно изучить. В тестовой группе показатель дохода распределен более равномерно и варьеруется от 2000 до 4000.

5. Для оценки результатов последнего прошедшего тематического события в игре сначала посмотрим на метрики привлечения. Какое количество постоянно играющих пользователей (игроков) участвовало в тематическом событии? Посмотрим на средний достигаемый уровень игроков в событии (по дням). Можем сравнить с другими ежемесячными событиями, сделав когортный анализ, посчитав достигаемый уровень игроков по дням и построив heatmap. Таким образом, мы можем оценить скорость получения наград игроками (на сколько сложно проходить уровни). Рассчитав DAU (ежедневные активные пользователи) в течение события и MAU (ежемесячные активные пользователи), можем увидеть, как увеличился приток активных пользователей (игроков). Также можно рассчитать количество сессий в день и среднюю длину сессии на одного пользователя (по событиям). Рассмотреть длину игровой сессии бывает полезно, чтобы понаблюдать, какой процент игровых сессий длится менее Х минут, и какое количество сессий длится более Х минут. Т.е. отслеживать, какой процент пользователей играет долгие периоды времени, в сравнении с теми, кто не надолго заходит в игру.
