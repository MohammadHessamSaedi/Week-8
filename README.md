این پروژه یک شبیه‌سازی کامل از سیستم Cyclic Prefix OFDM (CP-OFDM) را ارائه می‌دهد و مراحل آن به شرح زیر است:
تولید داده‌های تصادفی و مدولاسیون آن‌ها با QPSK به سمبل‌های پیچیده.
تقسیم سمبل‌ها به بلوک‌های ۶۴تایی و اعمال IFFT برای تولید سیگنال OFDM در حوزه زمان.
افزودن Cyclic Prefix (CP) به هر بلوک برای جلوگیری از تداخل سمبل‌ها (ISI).
عبور سیگنال از کانال؛ کانال می‌تواند AWGN ساده یا کانال چندمسیری با تأخیرهای مختلف باشد.
افزودن نویز با SNR مشخص برای شبیه‌سازی شرایط واقعی.
حذف CP در گیرنده و بازگرداندن سیگنال به حوزه فرکانس با FFT.
محاسبه پاسخ فرکانسی کانال و اعمال اکوالایزر ZF یا MMSE برای حذف اثر کانال.
دیمودولاسیون سمبل‌ها به بیت‌های بازیابی‌شده با نزدیک‌سازی به نقاط کانستلیشن QPSK.
مقایسه بیت‌های بازیابی‌شده با بیت‌های اصلی و محاسبه BER (Bit Error Rate).
نمایش نمودارهای BER در مقابل SNR و کانستلیشن سمبل‌ها برای ارزیابی عملکرد سیستم.
import numpy as np
import matplotlib.pyplot as plt
from scipy.signal import fftconvolve
from numpy.fft import fft, ifft
from tqdm import trange

# -----------------------------------------
# QPSK (4-QAM) Modulation
# -----------------------------------------
def qam_mod(bits, M=4):
    """Map bits to M-QAM symbols. For M=4 -> QPSK (Gray mapped)."""
    k = int(np.log2(M))
    assert bits.size % k == 0, "Number of bits must be multiple of log2(M)"

    symbols = bits.reshape((-1, k))

    if M == 4:
        # Gray mapping for QPSK
        mapping = {
            (0,0): 1+1j,
            (0,1): -1+1j,
            (1,1): -1-1j,
            (1,0): 1-1j
        }
        sym = np.array([mapping[tuple(b)] for b in symbols]) / np.sqrt(2)
        return sym

    else:
        raise NotImplementedError("Only QPSK (M=4) is implemented.")


# -----------------------------------------
# QPSK Demodulation (Hard Decision)
# -----------------------------------------
def qam_demod(symbols, M=4):
    """Demap QAM symbols to bits (hard decision)."""
    if M != 4:
        raise NotImplementedError("Only QPSK (M=4) is implemented.")

    s = symbols * np.sqrt(2)
    bits = []

    for x in s:
        real = x.real
        imag = x.imag

        if real > 0 and imag > 0:
            bits.extend([0, 0])
        elif real < 0 and imag > 0:
            bits.extend([0, 1])
        elif real < 0 and imag < 0:
            bits.extend([1, 1])
        else:
            bits.extend([1, 0])

    return np.array(bits, dtype=np.uint8)
بخش اول — وارد کردن کتابخانه‌ها

import numpy as np

وارد کردن NumPy (کتابخانهٔ محاسبات عددی) و اختصاص نام مستعار np برای کار با آرایه‌ها و عملگرهای برداری.

import matplotlib.pyplot as plt

وارد کردن matplotlib برای رسم نمودارها (در این بخش فقط آماده‌است؛ خود توابع رسم در کد بالا استفاده نشده‌اند ولی در نوت‌بوک اصلی به‌کار می‌روند).

from scipy.signal import fftconvolve

وارد کردن fftconvolve از SciPy برای انجام کانولوشن سریع با FFT (برای مدل‌سازی کانال multipath در بخش‌های دیگر نوت‌بوک).

from numpy.fft import fft, ifft

وارد کردن توابع FFT و IFFT از NumPy برای تبدیل فرکانس ↔ زمان (در بخش OFDM).

from tqdm import trange

وارد کردن trange برای نشان‌دادن نوار پیشرفت (progress bar) در حلقه‌های طولانی — مفید در شبیه‌سازی که چندین تکرار دارد.
بخش دوم — تابع مدولاسیون qam_mod

def qam_mod(bits, M=4):

تعریف تابعی به نام qam_mod که دو ورودی دارد: bits (آرایه بیتی 0/1) و تعداد حالت‌های مدولاسیون M (پیش‌فرض 4 یعنی QPSK).

"""Map bits to M-QAM symbols. For M=4 -> QPSK (Gray mapped)."""

docstring کوتاه که هدف تابع را توضیح می‌دهد.

k = int(np.log2(M))

محاسبهٔ تعداد بیت‌هایی که هر سمبل می‌برد. برای QPSK (M=4) مقدار k = 2 است (هر سمبل از 2 بیت تشکیل می‌شود).

assert bits.size % k == 0, "Number of bits must be multiple of log2(M)"

بررسی ورودی: تعداد بیت‌ها باید مضربِ k باشد؛ در غیر اینصورت پیام خطا داده می‌شود.

symbols = bits.reshape((-1, k))

آرایهٔ بیتی را به سطرهایی با طول k تقسیم می‌کند؛ هر سطر نشان‌دهندهٔ بیت‌های یک سمبل است. -1 یعنی خود NumPy تعداد سطرها را تعیین می‌کند.

if M == 4:

شاخه‌ای مخصوص QPSK؛ پیاده‌سازی عمومی‌تر QAM در این نسخه انجام نشده.

بخش mapping (چند خط):

mapping = {
    (0,0): 1+1j,
    (0,1): -1+1j,
    (1,1): -1-1j,
    (1,0): 1-1j
}
نگاشت بیت‌ها به نقاط کانستلاسیون QPSK با Gray mapping. هر زوج بیت به یک عدد مختلط متناظر تبدیل می‌شود:

(0,0) → +1 + j

(0,1) → -1 + j

(1,1) → -1 - j

(1,0) → +1 - j

Gray mapping یعنی بیت‌های همسایه در کانستلاسیون تنها یک بیت اختلاف دارند — این باعث کاهش خطای بیت در نزدیکی مرزها می‌شود.

sym = np.array([mapping[tuple(b)] for b in symbols]) / np.sqrt(2)

برای هر ردیف symbols (هر دو بیت) نقطهٔ متناظر را از دیکشنری می‌گیرد و آرایه‌ای از سمبل‌های پیچیده می‌سازد. سپس تقسیم بر sqrt(2) انجام می‌شود تا توان متوسط سمبل‌ها نرمال شود (توان هر سمبل برابر 1 شود).

return sym

خروجی: آرایهٔ سمبل‌های پیچیده (یک‌بعدی).

else: raise NotImplementedError("Only QPSK (M=4) is implemented.")

اگر کاربر خواستهٔ M ≠ 4 بدهد، خطا می‌زند و اطلاع می‌دهد که فقط QPSK پشتیبانی شده.

بخش سوم — تابع دیمودولاسیون qam_demod

def qam_demod(symbols, M=4):

تعریف تابع دیمودولاسیون که ورودی symbols (آرایه سمبل‌های پیچیده دریافتی) و M دارد.

"""Demap QAM symbols to bits (hard decision)."""

توضیح کوتاه: دیمپ به بیت‌ها با تصمیم‌گیری سخت (hard decision).

if M != 4: raise NotImplementedError("Only QPSK (M=4) is implemented.")

بررسی ورودی؛ اگر M بینهایت یا غیرپشتیبانی‌شده باشد خطا می‌دهد.

s = symbols * np.sqrt(2)

ضریب نرمال‌سازی را بازمی‌گردانیم (مکرراً سمبل‌ها در مدولاتور بر sqrt(2) تقسیم شده بودند) تا تصمیم‌گیری راحت‌تر انجام شود؛ با ضرب در sqrt(2) کانستلاسیون به مقادیر اصلی ±1±j بازمی‌گردد.

bits = []

لیست خالی برای جمع‌آوری بیت‌های خروجی.

for x in s:

حلقه روی همهٔ سمبل‌های دریافتی (هر x یک عدد مختلط است).

real = x.real و imag = x.imag

استخراج مولفهٔ حقیقی و موهومی سمبل برای تعیین کوادرانت (ربع) کانستلاسیون.

23–30. شرط‌ها برای تصمیم‌گیری:
python if real > 0 and imag > 0: bits.extend([0, 0]) elif real < 0 and imag > 0: bits.extend([0, 1]) elif real < 0 and imag < 0: bits.extend([1, 1]) else: bits.extend([1, 0])
- بر اساس علامت مولفهٔ حقیقی و موهومی تعیین می‌کنیم سمبل در کدام کوادرانت قرار دارد و بیت‌های متناظر را اضافه می‌کنیم. این همان معکوسِ نگاشت Gray بالا است.

return np.array(bits, dtype=np.uint8)

تبدیل لیست بیت‌ها به آرایهٔ NumPy از نوع بایت unsigned و بازگرداندن آن به عنوان خروجی تابع.

نکات تکمیلی (عملی و مفهومی)

Normalization: تقسیم بر sqrt(2) در مدولاتور و ضرب در sqrt(2) در دمدولاتور برای این است که توان متوسط سمبل‌ها برابر 1 شود؛ این کار باعث می‌شود محاسبهٔ SNR و نویز ساده‌تر و استاندارد باشد.

Hard decision: این دمدولاتور «تصمیم سخت» می‌گیرد (بیت‌ها دقیقاً 0 یا 1). می‌توان نسخهٔ «تصمیم نرم» (soft) ساخت تا برای دیکودرهای کانال (مثل LDPC/ Turbo) مفید باشد.

Gray mapping: انتخاب نگاشت Gray کاهش احتمال خطای بیت را در صورت خطای سمبل (transition بین نزدیک‌ترین نقاط) تضمین می‌کند.

گسترش به M>4: پیاده‌سازی عمومی QAM نیازمند نگاشت پیچیده‌تر (ماتریس نقاط شبکه‌ای) و تبدیل بیت→نماد منطقی‌تر است؛ برای این نوت‌بوک QPSK کفایت می‌کند.
