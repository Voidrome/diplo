# Проверка сроков сдачи книг
def check_due_dates():
    now = datetime.now()
    
    # Проверка за 3 дня до сдачи
    due_date_3_days = (now + timedelta(days=3)).strftime('%d.%m.%Y')
    cursor.execute('''
    SELECT b.title, b.reader_surname, b.reader_phone, b.due_date
    FROM books b
    JOIN users u ON b.reader_phone = u.phone
    WHERE b.due_date = ? AND u.chat_id IS NOT NULL
    ''', (due_date_3_days,))
    books_3_days = cursor.fetchall()
    
    for book in books_3_days:
        title, surname, phone, due_date = book
        cursor.execute('SELECT chat_id FROM users WHERE phone = ?', (phone,))
        result = cursor.fetchone()
        if result and result[0]:
            reader_chat_id = result[0]
            bot.send_message(reader_chat_id, 
                           f'Напоминание: книгу "{title}" нужно сдать через 3 дня (до {due_date})!')
    
    # Проверка за 1 день до сдачи
    due_date_1_day = (now + timedelta(days=1)).strftime('%d.%m.%Y')
    cursor.execute('''
    SELECT b.title, b.reader_surname, b.reader_phone, b.due_date
    FROM books b
    JOIN users u ON b.reader_phone = u.phone
    WHERE b.due_date = ? AND u.chat_id IS NOT NULL
    ''', (due_date_1_day,))
    books_1_day = cursor.fetchall()
    
    for book in books_1_day:
        title, surname, phone, due_date = book
        cursor.execute('SELECT chat_id FROM users WHERE phone = ?', (phone,))
        result = cursor.fetchone()
        if result and result[0]:
            reader_chat_id = result[0]
            bot.send_message(reader_chat_id, 
                           f'Срочное напоминание: книгу "{title}" нужно сдать завтра (до {due_date})!')
    
    # Проверка просроченных книг (каждые 3 дня)
    today_str = now.strftime('%d.%m.%Y')
    cursor.execute('''
    SELECT book_number, title, reader_phone, due_date, last_reminder
    FROM books 
    WHERE due_date < ?
    ''', (today_str,))
    overdue_books = cursor.fetchall()
    
    for book in overdue_books:
        book_number, title, phone, due_date, last_reminder = book
        
        # Проверяем, когда было последнее напоминание
        reminder_date = None
        if last_reminder:
            try:
                reminder_date = datetime.strptime(last_reminder, '%Y-%m-%d').date()
            except:
                pass
        
        # Если напоминание не отправлялось или прошло 3 дня
        if not reminder_date or (now.date() - reminder_date).days >= 3:
            cursor.execute('SELECT chat_id FROM users WHERE phone = ?', (phone,))
            result = cursor.fetchone()
            if result and result[0]:
                reader_chat_id = result[0]
                bot.send_message(reader_chat_id, 
                               f'⏰ Просрочено! Книгу "{title}" нужно было сдать {due_date}!\n'
                               f'Пожалуйста, верните книгу как можно скорее!')
            
            # Обновляем дату последнего напоминания
            cursor.execute('UPDATE books SET last_reminder = ? WHERE book_number = ?', 
                          (now.strftime('%Y-%m-%d'), book_number))
            conn.commit()

# Отправка напоминаний о событиях
def send_event_reminders():
    now = datetime.now()
    # Напоминаем за 1 день до события
    reminder_date = (now + timedelta(days=1)).strftime('%d.%m.%Y')
    
    # Находим события, которые будут через 1 день
    cursor.execute('''
    SELECT id, title, date, time 
    FROM events 
    WHERE date = ? AND confirmed = 1
    ''', (reminder_date,))
    events = cursor.fetchall()
    
    for event in events:
        event_id, title, date, time = event
        # Находим всех, кто записался на событие и еще не получил напоминание
        cursor.execute('''
        SELECT u.chat_id, u.surname 
        FROM event_attendance ea
        JOIN users u ON ea.user_phone = u.phone
        WHERE ea.event_id = ? AND ea.reminder_sent = 0 AND u.chat_id IS NOT NULL
        ''', (event_id,))
        attendees = cursor.fetchall()
        
        for attendee in attendees:
            chat_id, surname = attendee
            try:
                bot.send_message(
                    chat_id,
                    f'🔔 Напоминание о событии!\n\n'
                    f'Вы записались на событие:\n'
                    f'📌 {title}\n'
                    f'📅 {date} в {time}\n\n'
                    f'До события остался 1 день!'
                )
                # Помечаем, что напоминание отправлено
                cursor.execute('''
                UPDATE event_attendance 
                SET reminder_sent = 1 
                WHERE event_id = ? AND user_phone = (
                    SELECT phone FROM users WHERE chat_id = ?
                )
                ''', (event_id, chat_id))
                conn.commit()
            except Exception as e:
                print(f"Ошибка отправки напоминания: {e}")

# Очистка старых данных
def clean_old_data():
    today = datetime.now().date()
    
    # Удаляем события, которые прошли более 3 дней назад
    three_days_ago = (today - timedelta(days=3))
    cursor.execute('SELECT id, date FROM events')
    events = cursor.fetchall()
    for event in events:
        event_id, date_str = event
        try:
            event_date = datetime.strptime(date_str, '%d.%m.%Y').date()
            if event_date < three_days_ago:
                cursor.execute('DELETE FROM events WHERE id = ?', (event_id,))
                # Также удаляем записи о посещении
                cursor.execute('DELETE FROM event_attendance WHERE event_id = ?', (event_id,))
        except:
            continue
    conn.commit()
    
    # Удаляем новые поступления старше 1 месяца
    one_month_ago = (today - timedelta(days=30))
    cursor.execute('SELECT id, added_date FROM new_books')
    new_books = cursor.fetchall()
    for book in new_books:
        book_id, added_date_str = book
        try:
            added_date = datetime.strptime(added_date_str, '%Y-%m-%d %H:%M:%S').date()
            if added_date < one_month_ago:
                cursor.execute('DELETE FROM new_books WHERE id = ?', (book_id,))
        except:
            continue
    conn.commit()

# Запуск периодических задач (в 15:00)
def schedule_tasks():
    while True:
        now = datetime.now()
        
        # Проверка в 15:00 каждый день
        if now.hour == 15 and now.minute == 0:
            check_due_dates()
            send_event_reminders()
            clean_old_data()
            print(f"Проверка сроков выполнена в {now}")
        
        time.sleep(60)

thread = threading.Thread(target=schedule_tasks)
thread.daemon = True
thread.start()
