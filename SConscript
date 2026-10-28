from building import *

cwd = GetCurrentDir()
src = ['ws2812.c']
libm = []

if GetDepend(['WS2812B_TIMING_TABLE']) or GetDepend(['WS2812B_TIMING_SEARCH']) or GetDepend(['WS2812B_CHECK_EXTENDED']):
    src += ['ws2812_timing.c']

if GetDepend(['WS2812B_GAMMA_COMPUTE']):
    src += ['gamma/ws2812_gamma_compute.c']
    libm = ['m']

CPPPATH = [cwd, cwd + '/gamma']
group = DefineGroup('ws2812b', src, depend=['PKG_USING_WS2812B'], CPPPATH=CPPPATH, LIBS=libm)
Return('group')
