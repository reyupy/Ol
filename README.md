import asyncio
import os
import time
import signal
from contextlib import suppress
from docx import Document
from docx.shared import Pt, RGBColor, Cm
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.enum.style import WD_STYLE_TYPE
from reportlab.pdfgen.canvas import Canvas
from reportlab.lib.pagesizes import A4
from reportlab.pdfbase import pdfmetrics
from reportlab.pdfbase.ttfonts import TTFont
from reportlab.lib.units import cm
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph
from reportlab.lib import colors
from reportlab.lib.styles import getSampleStyleSheet, ParagraphStyle
from reportlab.pdfbase.pdfmetrics import registerFontFamily

from aiogram import Bot, Dispatcher, F
from aiogram.filters import Command, StateFilter
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import Message, FSInputFile, CallbackQuery
from dotenv import load_dotenv
from aiogram.types import Message, ReplyKeyboardMarkup, KeyboardButton
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton, WebAppInfo
from aiogram.utils.keyboard import InlineKeyboardBuilder
from base64 import b64decode
from datetime import datetime
from io import BytesIO

import requests
from dotenv import load_dotenv
from PIL import Image
import logging
import speech_recognition as sr
from pydub import AudioSegment
import subprocess

# Настройка логирования
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('bot.log'),
        logging.StreamHandler()  # Добавляем вывод в консоль
    ]
)
logger = logging.getLogger(__name__)


def create_KTP(data):
    logger.info("Начало создания КТП")
    try:
        load_dotenv()
        folder_id = os.getenv("YANDEX_FOLDER_ID")
        api_key = os.getenv("YANDEX_API_KEY")

        if not folder_id or not api_key:
            logger.error("Отсутствуют переменные окружения YANDEX_FOLDER_ID или YANDEX_API_KEY")
            return None

        logger.info("Отправка запроса к Yandex API")
        gpt_model = 'yandexgpt-lite'

        system_prompt = 'Ты помощник для создания документов на основе имеющихся данных. Напиши данный далее документ по современной школьной программе ФГОС по имеющимся данным.'
        user_prompt = data

        body = {
            'modelUri': f'gpt://{folder_id}/{gpt_model}',
            'completionOptions': {'stream': False, 'temperature': 0.3, 'maxTokens': 2000},
            'messages': [
                {'role': 'system', 'text': system_prompt},
                {'role': 'user', 'text': user_prompt},
            ],
        }
        url = 'https://llm.api.cloud.yandex.net/foundationModels/v1/completionAsync'
        headers = {
            'Content-Type': 'application/json',
            'Authorization': f'Api-Key {api_key}'
        }

        response = requests.post(url, headers=headers, json=body)
        operation_id = response.json().get('id')

        url = f"https://llm.api.cloud.yandex.net:443/operations/{operation_id}"
        headers = {"Authorization": f"Api-Key {api_key}"}

        while True:
            response = requests.get(url, headers=headers)
            done = response.json()["done"]
            if done:
                break
            time.sleep(2)

        data = response.json()
        answer = data['response']['alternatives'][0]['message']['text']

        logger.info("КТП успешно создан")
        return answer
    except Exception as e:
        logger.error(f"Ошибка при создании КТП: {e}")
        return None

# Определение состояний бота
class UserStates(StatesGroup):
    waiting_for_token = State()  # Ожидание ввода токена
    role_selection = State()  # Выбор роли
    document_selection = State()  # Выбор документа
    document_creation = State()  # Создание документа
    # Добавляем новые состояния для создания рабочей программы
    work_program_subject = State()
    work_program_class = State()
    work_program_hours = State()
    work_program_days = State()  # Добавляем новое состояние для выбора дней
    work_program_goals = State()
    # Добавляем новые состояния для календарно-тематического планирования
    calendar_plan_subject = State()
    calendar_plan_class = State()
    calendar_plan_hours = State()
    calendar_plan_days = State()
    calendar_plan_topics = State()
    # Добавляем состояния для характеристики ученика
    student_name = State()
    student_class = State()
    student_birth_date = State()
    student_characteristics = State()
    # Состояния для итогового собеседования
    final_interview_class = State()  # Новое состояние для выбора класса
    final_interview_students = State()


def create_table(document, headers, rows, style='Table Grid'):
    cols_number = len(headers)
    table = document.add_table(rows=1, cols=cols_number)
    table.style = style

    # Заполняем заголовки
    hdr_cells = table.rows[0].cells
    for i in range(cols_number):
        paragraph = hdr_cells[i].paragraphs[0]
        paragraph.alignment = WD_ALIGN_PARAGRAPH.CENTER
        run = paragraph.add_run(headers[i])
        run.font.name = 'Times New Roman'
        run.font.size = Pt(12)
        run.font.bold = True

    # Заполняем данные
    for row in rows:
        row_cells = table.add_row().cells
        for i in range(cols_number):
            paragraph = row_cells[i].paragraphs[0]
            paragraph.alignment = WD_ALIGN_PARAGRAPH.CENTER
            run = paragraph.add_run(str(row[i]))
            run.font.name = 'Times New Roman'
            run.font.size = Pt(12)

    return table


# Класс для работы с данными (имитация базы данных)
class Database:
    def __init__(self):
        self.schools_data = {
            "t 1": {
                "school_name": "МОБУ СОШ №1",
                "school_address": "г. Якутск, ул. Ленина, 1"
            },
            "teacher 2": {
                "school_name": "МОБУ СОШ №2",
                "school_address": "г. Якутск, ул. Пушкина, 2"
            },
            "teacher 3": {
                "school_name": "МОБУ Гимназия №3",
                "school_address": "г. Якутск, ул. Гоголя, 3"
            }
        }
        # Словарь для хранения созданных документов
        self.created_documents = {}

        # Загружаем шрифт FreeSans
        self._setup_fonts()

    def _setup_fonts(self):
        try:
            # Используем встроенные шрифты ReportLab без регистрации
            logger.info("Настройка шрифтов для PDF...")
            # Helvetica и Helvetica-Bold уже встроены в ReportLab
            # и не требуют регистрации
            logger.info("Шрифты успешно настроены")
        except Exception as e:
            logger.error(f"Ошибка при настройке шрифтов: {e}")
            pass

    def verify_token(self, token: str) -> bool:
        return token in self.schools_data

    def get_school_info(self, token: str) -> dict:
        return self.schools_data.get(token, {})

    def save_docx(self, document_name: str, content: str) -> str:
        try:
            # Создаем имя файла DOCX
            docx_filename = f"{document_name}.docx"

            # Создаем новый документ
            doc = Document()

            # Настраиваем поля страницы (в сантиметрах)
            sections = doc.sections
            for section in sections:
                section.left_margin = Cm(2)
                section.right_margin = Cm(2)
                section.top_margin = Cm(2)
                section.bottom_margin = Cm(2)

# Если это протокол итогового собеседования
            if "Итоговое_собеседование" in document_name:
                # Добавляем заголовок
                header = doc.add_paragraph()
                header.alignment = WD_ALIGN_PARAGRAPH.CENTER
                header_run = header.add_run("ПРОТОКОЛ\nитогового собеседования по русскому языку")
                header_run.font.name = 'Times New Roman'
                header_run.font.size = Pt(14)
                header_run.font.bold = True

                # Добавляем информацию о классе и школе
                class_info = doc.add_paragraph()
                class_info.alignment = WD_ALIGN_PARAGRAPH.CENTER
                class_run = class_info.add_run(content.split('\n')[0])
                class_run.font.name = 'Times New Roman'
                class_run.font.size = Pt(12)

                # Добавляем дату
                date_info = doc.add_paragraph()
                date_info.alignment = WD_ALIGN_PARAGRAPH.RIGHT
                current_date = datetime.now().strftime("%d.%m.%Y")
                date_run = date_info.add_run(f"Дата проведения: {current_date}")
                date_run.font.name = 'Times New Roman'
                date_run.font.size = Pt(12)

                # Создаем заголовки для таблицы
                headers = [
                    '№',
                    'ФИО',
                    'ИЧ\n(2б.)',
                    'ТЧ\n(1б.)',
                    'П\n(4б.)',
                    'Г\n(3б.)',
                    'О\n(3б.)',
                    'Р\n(3б.)',
                    'М\n(3б.)',
                    'Д\n(2б.)',
                    'Итого\n(20б.)',
                    'Зачёт'
                ]

                # Создаем строки с данными учеников
                students = content.split('\n')[1:]  # Пропускаем первую строку с информацией о классе
                rows = []
                for i, student in enumerate(students, 1):
                    if student.strip():  # Проверяем, что строка не пустая
                        # Создаем пустую строку для каждого ученика
                        row = [str(i), student.strip()]
                        row.extend([''] * 10)  # Добавляем 10 пустых ячеек для оценок
                        rows.append(row)

                # Создаем таблицу
                table = doc.add_table(rows=1, cols=len(headers))
                table.style = 'Table Grid'
                table.autofit = False

                # Заполняем заголовки
                for i, header in enumerate(headers):
                    cell = table.cell(0, i)
                    paragraph = cell.paragraphs[0]
                    paragraph.alignment = WD_ALIGN_PARAGRAPH.CENTER
                    run = paragraph.add_run(header)
                    run.font.name = 'Times New Roman'
                    run.font.size = Pt(10)
                    run.font.bold = True

                # Заполняем данные
                for row_data in rows:
                    cells = table.add_row().cells
                    for i, val in enumerate(row_data):
                        paragraph = cells[i].paragraphs[0]
                        paragraph.alignment = WD_ALIGN_PARAGRAPH.CENTER
                        run = paragraph.add_run(str(val))
                        run.font.name = 'Times New Roman'
                        run.font.size = Pt(10)

                # Устанавливаем ширину столбцов
                widths = [1, 4, 1, 1, 1, 1, 1, 1, 1, 1, 1.5, 1.5]  # Пропорции ширины столбцов
                for i, width in enumerate(widths):
                    for cell in table.columns[i].cells:
                        cell.width = Cm(width)

                # Добавляем подписи
                doc.add_paragraph()
                signatures = doc.add_paragraph()
                signatures.alignment = WD_ALIGN_PARAGRAPH.LEFT
                signatures_text = """
Экзаменатор-собеседник: _________ / ____________

Эксперт: _________ / ________________"""
                signatures_run = signatures.add_run(signatures_text)
                signatures_run.font.name = 'Times New Roman'
                signatures_run.font.size = Pt(12)

            else:
                # Для остальных документов оставляем стандартное форматирование
                paragraphs = content.split('\n')
                is_first = True
                for paragraph in paragraphs:
                    if paragraph.strip():
                        p = doc.add_paragraph()
                        if is_first:
                            p.style = 'CustomTitle'
                            is_first = False
                        elif ':' in paragraph and len(paragraph) < 50:
                            p.style = 'CustomHeading'
                        else:
                            p.style = 'CustomNormal'
                        p.add_run(paragraph.strip())

            # Сохраняем документ
            doc.save(docx_filename)
            return docx_filename

        except Exception as e:
            print(f"Ошибка при создании DOCX: {e}")
            return None

    def save_pdf(self, document_name: str, content: str, is_interview: bool = False,
                 interview_data: dict = None) -> str:
        try:
            # Создаем PDF документ
            pdf_filename = f"{document_name}.pdf"
            pdf_canvas = Canvas(pdf_filename, pagesize=A4)

            # Настраиваем базовые параметры
            y_position = A4[1] - 50  # Начинаем с верха страницы
            line_height = 20  # Высота строки
            margin = 50  # Отступ слева

            if is_interview:
                # Заголовок
                pdf_canvas.setFont('Helvetica-Bold', 14)
                pdf_canvas.drawString(margin, y_position, "ПРОТОКОЛ")
                y_position -= line_height
                pdf_canvas.drawString(margin, y_position, "итогового собеседования по русскому языку")
                y_position -= line_height * 2

                # Информация о классе и школе
                pdf_canvas.setFont('Helvetica', 12)
                pdf_canvas.drawString(margin, y_position, f"{interview_data['class']} класс")
                y_position -= line_height
                pdf_canvas.drawString(margin, y_position, interview_data['school'])
                y_position -= line_height * 2

                # Создаем таблицу
                headers = ['№', 'ФИО', 'ИЧ(2б.)', 'ТЧ(1б.)', 'П(4б.)', 'Г(3б.)', 'О(3б.)',
                           'Р(3б.)', 'М(3б.)', 'Д(2б.)', 'Итого(20б.)', 'Зачёт']

                # Рисуем заголовки таблицы
                x_position = margin
                cell_widths = [30, 200] + [50] * 10  # Ширины столбцов

                pdf_canvas.setFont('Helvetica-Bold', 10)
                for i, header in enumerate(headers):
                    pdf_canvas.rect(x_position, y_position, cell_widths[i], line_height)
                    pdf_canvas.drawString(x_position + 5, y_position + 5, header)
                    x_position += cell_widths[i]

                y_position -= line_height

                # Заполняем данные студентов
                pdf_canvas.setFont('Helvetica', 10)
                for i, student in enumerate(interview_data['students'], 1):
                    x_position = margin
                    # Номер
                    pdf_canvas.rect(x_position, y_position, cell_widths[0], line_height)
                    pdf_canvas.drawString(x_position + 5, y_position + 5, str(i))
                    x_position += cell_widths[0]

                    # ФИО
                    pdf_canvas.rect(x_position, y_position, cell_widths[1], line_height)
                    pdf_canvas.drawString(x_position + 5, y_position + 5, student)
                    x_position += cell_widths[1]

                    # Пустые ячейки для оценок
                    for width in cell_widths[2:]:
                        pdf_canvas.rect(x_position, y_position, width, line_height)
                        x_position += width

                    y_position -= line_height

# Добавляем подписи
                y_position -= line_height * 2
                pdf_canvas.setFont('Helvetica', 12)
                pdf_canvas.drawString(margin, y_position, "Экзаменатор-собеседник: _________ / ________________")
                y_position -= line_height * 2
                pdf_canvas.drawString(margin, y_position, "Эксперт: _________ / ________________")

            else:
                # Для обычных документов
                pdf_canvas.setFont('Helvetica', 12)
                for paragraph in content.split('\n'):
                    if paragraph.strip():
                        # Если текст слишком длинный, разбиваем его на строки
                        words = paragraph.split()
                        line = []
                        for word in words:
                            line.append(word)
                            # Проверяем ширину строки
                            if pdf_canvas.stringWidth(' '.join(line), 'Helvetica', 12) > A4[0] - margin * 2:
                                line.pop()  # Убираем последнее слово
                                pdf_canvas.drawString(margin, y_position, ' '.join(line))
                                y_position -= line_height
                                line = [word]
                        if line:
                            pdf_canvas.drawString(margin, y_position, ' '.join(line))
                            y_position -= line_height * 1.5

                        # Если достигли конца страницы, создаем новую
                        if y_position < 50:
                            pdf_canvas.showPage()
                            y_position = A4[1] - 50
                            pdf_canvas.setFont('Helvetica', 12)

            pdf_canvas.showPage()
            pdf_canvas.save()
            return pdf_filename

        except Exception as e:
            logger.error(f"Ошибка при создании PDF: {e}")
            return None


# Создание глобального диспетчера
dp = Dispatcher()


# Обработчик команды /start
@dp.message(Command("start"))
async def cmd_start(message: Message, state: FSMContext):
    await state.set_state(UserStates.waiting_for_token)
    await message.answer("Добро пожаловать! Пожалуйста, введите ваш токен доступа (t 1):")


# Обработчик проверки токена
@dp.message(StateFilter(UserStates.waiting_for_token))
async def check_token(message: Message, state: FSMContext):
    if db.verify_token(message.text):
        # Сохраняем токен и информацию о школе в состоянии
        school_info = db.get_school_info(message.text)
        await state.update_data(
            user_token=message.text,
            school_name=school_info["school_name"],
            school_address=school_info["school_address"]
        )

        # Если токен верный, показываем меню выбора роли
        kb = InlineKeyboardBuilder()
        kb.add(
            InlineKeyboardButton(text="Учитель-предметник", callback_data="subject_teacher"),
            InlineKeyboardButton(text="Классный руководитель", callback_data="class_teacher"),
            InlineKeyboardButton(text="Завуч", callback_data="head_teacher"),
            InlineKeyboardButton(text="Мои документы", callback_data="my_documents")
        )
        kb.adjust(1)
        await message.answer("Выберите вашу роль:", reply_markup=kb.as_markup())
        await state.set_state(UserStates.role_selection)
    else:
        await message.answer("Неверный токен. Попробуйте еще раз:")


# Обработчик выбора роли учителя-предметника
@dp.callback_query(F.data == "subject_teacher")
async def subject_teacher_menu(callback: CallbackQuery, state: FSMContext):
    # Создаем клавиатуру с типами документов
    kb = InlineKeyboardBuilder()
    kb.add(
        InlineKeyboardButton(text="Рабочая программа", callback_data="work_program"),
        InlineKeyboardButton(text="Календарно-тематическое планирование", callback_data="calendar_plan")
    )
    kb.adjust(1)
    await callback.message.answer("Выберите тип документа:", reply_markup=kb.as_markup())
    await callback.answer()

# Обработчик просмотра созданных документов
@dp.callback_query(F.data == "my_documents")
async def show_my_documents(callback: CallbackQuery, state: FSMContext):
    documents = db.created_documents

    if not documents:
        await callback.message.answer("У вас пока нет созданных документов.")
        return

    # Создаем клавиатуру со списком документов
    kb = InlineKeyboardBuilder()
    for doc_name in documents.keys():
        kb.add(InlineKeyboardButton(
            text=doc_name,
            callback_data=f"view_doc_{doc_name}"
        ))
    kb.adjust(1)

    # Добавляем кнопку возврата
    kb.add(InlineKeyboardButton(
        text="◀️ Назад",
        callback_data="back_to_menu"
    ))

    await callback.message.answer(
        "Ваши созданные документы:",
        reply_markup=kb.as_markup()
    )
    await callback.answer()


# Обработчик просмотра конкретного документа
@dp.callback_query(F.data.startswith("view_doc_"))
async def view_document(callback: CallbackQuery, state: FSMContext):
    # Получаем имя документа из callback_data
    doc_name = callback.data.replace("view_doc_", "")
    doc_content = db.created_documents.get(doc_name)

    if doc_content:
        await callback.message.answer(f"Содержание документа '{doc_name}':\n\n{doc_content}")
    else:
        await callback.message.answer("Документ не найден.")

    await callback.answer()


# Обработчик выбора рабочей программы
@dp.callback_query(F.data == "work_program")
async def handle_work_program(callback: CallbackQuery, state: FSMContext):
    kb = InlineKeyboardBuilder()
    kb.add(
        InlineKeyboardButton(text="Создать новую программу", callback_data="create_work_program"),
        InlineKeyboardButton(text="Загрузить шаблон (в разработке)", callback_data="load_work_template"),
        InlineKeyboardButton(text="◀️ Назад", callback_data="back_to_menu")
    )
    kb.adjust(1)
    await callback.message.answer("Выберите действие:", reply_markup=kb.as_markup())
    await callback.answer()


# Обработчик кнопки "Назад"
@dp.callback_query(F.data == "back_to_menu")
async def back_to_menu(callback: CallbackQuery, state: FSMContext):
    kb = InlineKeyboardBuilder()
    kb.add(
        InlineKeyboardButton(text="Учитель-предметник", callback_data="subject_teacher"),
        InlineKeyboardButton(text="Классный руководитель", callback_data="class_teacher"),
        InlineKeyboardButton(text="Завуч", callback_data="head_teacher"),
        InlineKeyboardButton(text="Мои документы", callback_data="my_documents")
    )
    kb.adjust(1)
    await callback.message.answer("Выберите вашу роль:", reply_markup=kb.as_markup())
    await callback.answer()


# Обработчик создания рабочей программы
@dp.callback_query(F.data == "create_work_program")
async def start_work_program_creation(callback: CallbackQuery, state: FSMContext):
    print('рабочая программа')
    await state.set_state(UserStates.work_program_subject)
    await callback.message.answer(
        "Давайте создадим рабочую программу.\n"
        "Шаг 1: Укажите название предмета:"
    )
    await callback.answer()


# Обработчик ввода предмета
@dp.message(StateFilter(UserStates.work_program_subject))
async def process_subject(message: Message, state: FSMContext):
    await state.update_data(subject=message.text)
    await state.set_state(UserStates.work_program_class)
    await message.answer(
        "Шаг 2: Выберите класс:"
    )


# Обработчик ввода класса
@dp.message(StateFilter(UserStates.work_program_class))
async def process_class(message: Message, state: FSMContext):
    await state.update_data(grade=message.text)
    await state.set_state(UserStates.work_program_hours)
    await message.answer(
        "Шаг 3: Укажите необходимое количество часов в год:"
    )


# Обработчик ввода количества часов
@dp.message(StateFilter(UserStates.work_program_hours))
async def process_hours(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("Пожалуйста, введите число.")
        return

await state.update_data(hours=int(message.text))
    await state.set_state(UserStates.work_program_days)

    # Создаем клавиатуру для выбора количества учебных дней
    kb = InlineKeyboardBuilder()
    kb.add(
        InlineKeyboardButton(text="5 дней", callback_data="days_5"),
        InlineKeyboardButton(text="6 дней", callback_data="days_6")
    )
    kb.adjust(2)

    await message.answer(
        "Шаг 4: Выберите количество учебных дней в неделю:",
        reply_markup=kb.as_markup()
    )


# Добавляем обработчик выбора количества дней
@dp.callback_query(F.data.startswith("days_"))
async def process_days_selection(callback: CallbackQuery, state: FSMContext):
    days = callback.data.split("_")[1]
    await state.update_data(school_days=days)
    await state.set_state(UserStates.work_program_goals)

    await callback.message.answer(
        "Шаг 5: Опишите цели и задачи программы:"
    )
    await callback.answer()


# Обработчик ввода целей
@dp.message(StateFilter(UserStates.work_program_goals))
async def process_goals_and_create(message: Message, state: FSMContext):
    await state.update_data(goals=message.text)

    # Получаем все данные
    data = await state.get_data()

    try:
        # Формируем документ
        document_text = f"""РАБОЧАЯ ПРОГРАММА

Предмет: {data['subject']}
Класс: {data['grade']}
Количество часов: {data['hours']}
Учебная неделя: {data['school_days']}-дневная

ЦЕЛИ И ЗАДАЧИ:
{data['goals']}
"""

        # Добавляем шаблон, если он есть
        user_token = data.get("user_token")
        template = db.get_template(user_token, "subject_teacher")
        if template:
            document_text += f"\nДОПОЛНИТЕЛЬНАЯ ИНФОРМАЦИЯ ИЗ ШАБЛОНА:\n{template}"

        # Генерируем уникальное имя документа
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        document_name = f"Рабочая_программа_{data['subject']}_{data['grade']}_{timestamp}"

        # Сохраняем документ
        db.created_documents[document_name] = document_text

        # Создаем клавиатуру для возврата
        kb = InlineKeyboardBuilder()
        kb.add(InlineKeyboardButton(text="◀️ В главное меню", callback_data="back_to_menu"))
        kb.add(InlineKeyboardButton(text="📄 Мои документы", callback_data="my_documents"))
        kb.adjust(1)

        await message.answer(
            f"✅ Рабочая программа успешно создана!\n"
            f"📝 Название документа: {document_name}",
            reply_markup=kb.as_markup()
        )

        # Возвращаемся к выбору роли, сохраняя токен
        await state.clear()
        await state.set_state(UserStates.role_selection)
        await state.update_data(user_token=user_token)

    except KeyError as e:
        # Если какие-то данные отсутствуют
        await message.answer(
            "❌ Произошла ошибка при создании документа. "
            "Пожалуйста, начните процесс создания заново."
        )
        await state.clear()
        await cmd_start(message, state)
    except Exception as e:
        # Если произошла другая ошибка
        await message.answer(
            "❌ Произошла непредвиденная ошибка. "
            "Пожалуйста, попробуйте еще раз."
        )
        await state.clear()
        await cmd_start(message, state)


# Обработчик создания календарно-тематического плана
@dp.callback_query(F.data == "calendar_plan")
async def start_calendar_plan_creation(callback: CallbackQuery, state: FSMContext):
    print('календарь')
    await state.set_state(UserStates.calendar_plan_subject)
    await callback.message.answer(
        "Давайте создадим календарно-тематическое планирование.\n"
        "Шаг 1: Укажите название предмета:"
    )
    await callback.answer()


# Обработчик ввода предмета для КТП
@dp.message(StateFilter(UserStates.calendar_plan_subject))
async def process_calendar_subject(message: Message, state: FSMContext):
    await state.update_data(subject=message.text)
    await state.set_state(UserStates.calendar_plan_class)
    await message.answer(
        "Шаг 2: Выберите класс:"
    )

# Обработчик ввода класса для КТП
@dp.message(StateFilter(UserStates.calendar_plan_class))
async def process_calendar_class(message: Message, state: FSMContext):
    await state.update_data(grade=message.text)
    await state.set_state(UserStates.calendar_plan_hours)
    await message.answer(
        "Шаг 3: Укажите необходимое количество часов в год:"
    )


# Обработчик ввода количества часов для КТП
@dp.message(StateFilter(UserStates.calendar_plan_hours))
async def process_calendar_hours(message: Message, state: FSMContext):
    if not message.text.isdigit():
        await message.answer("Пожалуйста, введите число.")
        return

    await state.update_data(hours=int(message.text))
    await state.set_state(UserStates.calendar_plan_days)

    kb = InlineKeyboardBuilder()
    kb.add(
        InlineKeyboardButton(text="5 дней", callback_data="calendar_days_5"),
        InlineKeyboardButton(text="6 дней", callback_data="calendar_days_6")
    )
    kb.adjust(2)

    await message.answer(
        "Шаг 4: Выберите количество учебных дней в неделю:",
        reply_markup=kb.as_markup()
    )


# Обработчик выбора количества дней для КТП
@dp.callback_query(F.data.startswith("calendar_days_"))
async def process_calendar_days_selection(callback: CallbackQuery, state: FSMContext):
    print(2)
    days = callback.data.split("_")[2]
    await state.update_data(school_days=days)
    await state.set_state(UserStates.calendar_plan_topics)

    await callback.message.answer(
        "Шаг 5: Введите темы занятий (каждая тема с новой строки):"
    )
    print(1)
    await callback.answer()


# Обработчик ввода тем и создания КТП
@dp.message(StateFilter(UserStates.calendar_plan_topics))
async def process_calendar_topics_and_create(message: Message, state: FSMContext):
    await state.update_data(topics=message.text)

    # Получаем все данные
    data = await state.get_data()

    try:
        # Формируем документ
        document_text = f"""КАЛЕНДАРНО-ТЕМАТИЧЕСКОЕ ПЛАНИРОВАНИЕ

Предмет: {data['subject']}
Класс: {data['grade']}
Количество часов: {data['hours']}
Учебная неделя: {data['school_days']}-дневная

ТЕМЫ ЗАНЯТИЙ:
{data['topics']}
"""
        document_text = create_KTP(document_text)
        print(document_text)

        # Генерируем уникальное имя документа
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        document_name = f"КТП_{data['subject']}_{data['grade']}_{timestamp}"

        # Сохраняем документ в базу данных
        db.created_documents[document_name] = document_text

        # Создаем DOCX файл
        docx_path = db.save_docx(document_name, document_text)

        if docx_path:
            # Создаем клавиатуру для возврата
            kb = InlineKeyboardBuilder()
            kb.add(InlineKeyboardButton(text="◀️ В главное меню", callback_data="back_to_menu"))
            kb.add(InlineKeyboardButton(text="📄 Мои документы", callback_data="my_documents"))
            kb.adjust(1)

            # Отправляем DOCX файл
            doc = FSInputFile(docx_path)
            await message.answer_document(
                doc,
                caption=f"✅ Календарно-тематическое планирование успешно создано!\n📝 Название документа: {document_name}",
                reply_markup=kb.as_markup()
            )

            # Удаляем временный DOCX файл
            try:
                os.remove(docx_path)
            except:
                pass
        else:
            await message.answer(
                "❌ Произошла ошибка при создании DOCX файла.\n"
                "Документ сохранен в текстовом формате.",
                reply_markup=kb.as_markup()
            )

        # Возвращаемся к выбору роли
        await state.clear()
        await state.set_state(UserStates.role_selection)

    except Exception as e:
        print(f"Error: {e}")
        await message.answer(
            "❌ Произошла ошибка при создании документа. "
            "Пожалуйста, попробуйте еще раз."
        )
        await state.clear()
        await cmd_start(message, state)

# Обработчик выбора роли классного руководителя
@dp.callback_query(F.data == "class_teacher")
async def class_teacher_menu(callback: CallbackQuery, state: FSMContext):
    # Создаем клавиатуру с типами документов
    kb = InlineKeyboardBuilder()
    kb.add(
        InlineKeyboardButton(text="Характеристика ученика", callback_data="student_characteristic"),
        InlineKeyboardButton(text="◀️ Назад", callback_data="back_to_menu")
    )
    kb.adjust(1)
    await callback.message.answer("Выберите тип документа:", reply_markup=kb.as_markup())
    await callback.answer()


# Обработчик создания характеристики ученика
@dp.callback_query(F.data == "student_characteristic")
async def start_student_characteristic(callback: CallbackQuery, state: FSMContext):
    await state.set_state(UserStates.student_name)
    await callback.message.answer(
        "Давайте создадим характеристику ученика.\n"
        "Шаг 1: Введите ФИО ученика:"
    )
    await callback.answer()


# Обработчик ввода имени ученика
@dp.message(StateFilter(UserStates.student_name))
async def process_student_name(message: Message, state: FSMContext):
    await state.update_data(student_name=message.text)
    await state.set_state(UserStates.student_class)
    await message.answer("Шаг 2: Укажите класс ученика:")


# Обработчик ввода класса ученика
@dp.message(StateFilter(UserStates.student_class))
async def process_student_class(message: Message, state: FSMContext):
    await state.update_data(student_class=message.text)
    await state.set_state(UserStates.student_birth_date)
    await message.answer("Шаг 3: Укажите дату рождения ученика (дд.мм.гггг):")


# Обработчик ввода даты рождения
@dp.message(StateFilter(UserStates.student_birth_date))
async def process_student_birth_date(message: Message, state: FSMContext):
    await state.update_data(student_birth_date=message.text)
    await state.set_state(UserStates.student_characteristics)
    await message.answer(
        "Шаг 4: Введите информацию об ученике:\n"
        "• Адрес проживания\n"
        "• Дата зачисления в школу и предыдущее место учебы\n"
        "• Информация о семье (родители, братья/сестры)\n"
        "• Успеваемость и способности к обучению\n"
        "• Любимые предметы\n"
        "• Участие в общественной жизни\n"
        "• Личные качества\n"
        "• Увлечения\n"
        "• Дополнительная информация"
    )


# Обработчик ввода характеристик и создания документа
@dp.message(StateFilter(UserStates.student_characteristics))
async def process_student_characteristics(message: Message, state: FSMContext):
    await state.update_data(characteristics=message.text)
    data = await state.get_data()

    try:
        # Формируем документ для отправки в ИИ
        ai_prompt = f"""На основе предоставленной информации сгенерируй характеристику ученика по следующему шаблону:

Характеристика
на обучающегося {data['student_class']} класса {data['school_name']}
{data['student_name']}, {data['student_birth_date']} г.р.,

Информация об ученике:
{data['characteristics']}

Требования к характеристике:
1. Строго следовать формату шаблона
2. Использовать официально-деловой стиль
3. Включить всю предоставленную информацию в логичной последовательности
4. Добавить стандартную информацию о физическом и психическом развитии
5. Указать информацию о поведении и отношении к учебе
6. В конце добавить информацию об отсутствии нарушений устава школы
7. Сохранить нейтральный, объективный тон повествования"""

        # Получаем сгенерированный текст от ИИ
        logger.info("Отправка запроса на генерацию характеристики")
        generated_text = create_KTP(ai_prompt)

        if not generated_text:
            raise Exception("Не удалось сгенерировать характеристику")

        # Добавляем подпись
        document_text = f"{generated_text}\n\nКлассный руководитель: _____________________"

        # Генерируем уникальное имя документа
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        document_name = f"Характеристика_{data['student_name']}_{timestamp}"

# Создаем PDF файл
        pdf_filename = db.save_pdf(document_name, document_text)

        if pdf_filename:
            # Создаем клавиатуру для возврата
            kb = InlineKeyboardBuilder()
            kb.add(InlineKeyboardButton(text="◀️ В главное меню", callback_data="back_to_menu"))
            kb.add(InlineKeyboardButton(text="📄 Мои документы", callback_data="my_documents"))
            kb.adjust(1)

            # Отправляем PDF файл
            doc_file = FSInputFile(pdf_filename)
            await message.answer_document(
                doc_file,
                caption=f"✅ Характеристика ученика успешно создана!\n📝 Название документа: {document_name}",
                reply_markup=kb.as_markup()
            )

            # Удаляем временный PDF файл
            try:
                os.remove(pdf_filename)
            except:
                pass
        else:
            await message.answer(
                "❌ Произошла ошибка при создании PDF файла.",
                reply_markup=kb.as_markup()
            )

        # Возвращаемся к выбору роли
        await state.clear()
        await state.set_state(UserStates.role_selection)

    except Exception as e:
        logger.error(f"Ошибка при создании характеристики: {e}")
        await message.answer(
            "❌ Произошла ошибка при создании документа. "
            "Пожалуйста, попробуйте еще раз."
        )
        await state.clear()
        await cmd_start(message, state)


# Обработчик выбора роли завуча
@dp.callback_query(F.data == "head_teacher")
async def head_teacher_menu(callback: CallbackQuery, state: FSMContext):
    # Создаем клавиатуру с типами документов
    kb = InlineKeyboardBuilder()
    kb.add(
        InlineKeyboardButton(text="Итоговое собеседование", callback_data="final_interview"),
        InlineKeyboardButton(text="◀️ Назад", callback_data="back_to_menu")
    )
    kb.adjust(1)
    await callback.message.answer("Выберите тип документа:", reply_markup=kb.as_markup())
    await callback.answer()


# Обработчик для итогового собеседования
@dp.callback_query(F.data == "final_interview")
async def start_final_interview(callback: CallbackQuery, state: FSMContext):
    await state.set_state(UserStates.final_interview_class)
    await callback.message.answer(
        "Укажите класс для итогового собеседования (например: 9А):"
    )
    await callback.answer()


# Обработчик выбора класса для итогового собеседования
@dp.message(StateFilter(UserStates.final_interview_class))
async def process_final_interview_class(message: Message, state: FSMContext):
    await state.update_data(interview_class=message.text)
    await state.set_state(UserStates.final_interview_students)
    await message.answer(
        "Введите список учеников для итогового собеседования.\n"
        "Формат ввода (каждый ученик с новой строки):\n"
        "ФИО\n\n"
        "Пример:\n"
        "Иванов Иван Иванович\n"
        "Петров Петр Петрович"
    )


# Обработчик ввода списка учеников
@dp.message(StateFilter(UserStates.final_interview_students))
async def process_final_interview_students(message: Message, state: FSMContext):
    data = await state.get_data()
    interview_class = data.get('interview_class')
    students_data = message.text.strip().split('\n')
    school_name = data.get('school_name', '')

    # Создаем клавиатуру для возврата заранее
    kb = InlineKeyboardBuilder()
    kb.add(InlineKeyboardButton(text="◀️ В главное меню", callback_data="back_to_menu"))
    kb.add(InlineKeyboardButton(text="📄 Мои документы", callback_data="my_documents"))
    kb.adjust(1)

    try:
        # Генерируем уникальное имя документа
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        document_name = f"Итоговое_собеседование_{interview_class}_{timestamp}"

        # Создаем PDF файл
        interview_data = {
            'class': interview_class,
            'school': school_name,
            'students': [student.strip() for student in students_data if student.strip()]
        }

pdf_filename = db.save_pdf(document_name, "", is_interview=True, interview_data=interview_data)

        if pdf_filename:
            # Отправляем PDF файл
            doc_file = FSInputFile(pdf_filename)
            await message.answer_document(
                doc_file,
                caption=f"✅ Протокол итогового собеседования для {interview_class} класса успешно создан!\n📝 Название документа: {document_name}",
                reply_markup=kb.as_markup()
            )

            # Удаляем временный PDF файл
            try:
                os.remove(pdf_filename)
            except:
                pass
        else:
            await message.answer(
                "❌ Произошла ошибка при создании PDF файла.",
                reply_markup=kb.as_markup()
            )

        # Возвращаемся к выбору роли
        await state.clear()
        await state.set_state(UserStates.role_selection)

    except Exception as e:
        logger.error(f"Ошибка при создании протокола итогового собеседования: {e}")
        await message.answer(
            "❌ Произошла ошибка при создании документа. "
            "Пожалуйста, попробуйте еще раз.",
            reply_markup=kb.as_markup()
        )
        await state.clear()
        await cmd_start(message, state)


def check_api_availability():
    try:
        response = requests.get("https://llm.api.cloud.yandex.net/health")
        return response.status_code == 200
    except:
        return False


async def shutdown(dispatcher: Dispatcher, bot: Bot):
    logger.info("Завершение работы бота...")
    await dispatcher.storage.close()
    await bot.session.close()
    logger.info("Бот остановлен")


# Основная функция запуска бота
async def main() -> None:
    logger.info("Запуск бота")
    # Загружаем переменные окружения
    load_dotenv()
    bot_token = os.getenv("TELEGRAM_BOT_TOKEN")

    if not bot_token:
        logger.error("Отсутствует токен бота в переменных окружения")
        return

    # Создаем файл-блокировку
    pid_file = "bot.pid"
    if os.path.exists(pid_file):
        logger.error("Бот уже запущен")
        return

    try:
        # Записываем PID текущего процесса
        with open(pid_file, "w") as f:
            f.write(str(os.getpid()))

        # Создаем экземпляр бота
        bot = Bot(token=bot_token)
        logger.info("Бот успешно создан и готов к работе")

        # Настраиваем обработчик сигналов для корректного завершения
        async def on_shutdown(signum, frame):
            logger.info("Получен сигнал завершения")
            await shutdown(dp, bot)
            # Удаляем файл-блокировку
            with suppress(FileNotFoundError):
                os.remove(pid_file)

        for sig in (signal.SIGINT, signal.SIGTERM):
            signal.signal(sig, lambda s, f: asyncio.create_task(on_shutdown(s, f)))

        # Запускаем бота
        logger.info("Запуск поллинга")
        await dp.start_polling(bot, allowed_updates=dp.resolve_used_update_types())

    except Exception as e:
        logger.error(f"Произошла ошибка: {e}")
    finally:
        # Удаляем файл-блокировку при любом завершении
        with suppress(FileNotFoundError):
            os.remove(pid_file)


# Точка входа в программу
if name == "__main__":
    # Создаем экземпляр базы данных
    db = Database()

    try:
        # Запускаем бота через asyncio
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("Бот остановлен вручную")
    except Exception as e:
        logger.error(f"Критическая ошибка: {e}")
        print("Перезапуск бота...")
