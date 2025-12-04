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
