#! /usr/bin/env python
#  -*- coding: utf-8 -*-
#
# GUI module generated by PAGE version 4.18
#  in conjunction with Tcl version 8.6
#    Nov 24, 2018 12:26:37 PM +0200  platform: Windows NT

import sys
import tkinter as tk
import serial
from scipy import ndimage
import numpy as np
import time
import tkinter.ttk as ttk
from PIL import Image, ImageTk
from tkinter import filedialog
from itertools import product
import matplotlib.pyplot as plt
import binascii

import bluetooth
try:
    import Tkinter as tk
except ImportError:
    import tkinter as tk

try:
    import ttk
    py3 = False
except ImportError:
    import tkinter.ttk as ttk
    py3 = True

import CDMA_TX_support

def vp_start_gui():
    '''Starting point when module is the main routine.'''
    global val, w, root
    root = tk.Tk()
    CDMA_TX_support.set_Tk_var()
    top = mainPage (root)
    CDMA_TX_support.init(root, top)
    root.mainloop()

w = None
def create_mainPage(root, *args, **kwargs):
    '''Starting point when module is imported by another program.'''
    global w, w_win, rt
    rt = root
    w = tk.Toplevel (root)
    CDMA_TX_support.set_Tk_var()
    top = mainPage (w)
    CDMA_TX_support.init(w, top, *args, **kwargs)
    return (w, top)

def destroy_mainPage():
    global w
    w.destroy()
    w = None


##############functions for CDMA##################

def bitfield(n):
    return [1 if digit == '1' else 0 for digit in bin(n)[2:].zfill(8)]


def bin2num(bitlist):
    out = 0
    for bit in bitlist:
        out = (out << 1) | bit
    return out


# vector a is longer than b (will be a=data vector , b=walsh vector)
# def high_demantion_cross(a, b):
#     cross_resault = []
#     for i in a:
#         # print (np.dot(i,b))
#         cross_resault = np.concatenate((cross_resault, np.dot(i, b))).astype(int)
#     # print (cross_resault)
#     return cross_resault

#improved
def high_demantion_cross(a,b):
    cross_resault=np.array([])
    return (a.reshape((-1,1))*b).reshape(1,len(a)*len(b))[0]


def encoder_data(dataMat, w):
    encoded = np.array([])
    for i in range(0, len(dataMat)):
        encoded = np.append(encoded, high_demantion_cross(dataMat[i], w[i])).astype(int)
        # print(encoded)
    encoded = np.reshape(encoded, (len(w), (int)(len(encoded) / len(w))))
    encoded = encoded.sum(axis=0)
    return encoded


def convert_img2vector(img):
    temp = []
    vec = []
    # convert 2d matrix to vector
    temp = np.reshape(img, len(img) * len(img[0])).astype(int)

    for i in temp:
        vec = np.append(vec, bitfield(i)).astype(int)
    return vec

def add_awgn_noise(x,SNR_dB):
    #https://www.gaussianwaves.com/2015/06/how-to-generate-awgn-noise-in-matlaboctave-without-using-in-built-awgn-function/
    #y=awgn_noise(x,SNR) adds AWGN noise vector to signal 'x' to generate a
    #resulting signal vector y of specified SNR in dB
    #rng('default');#set the random generator seed to default (for comparison only)
    L=len(x);
    SNR = np.power(10,SNR_dB/10); #SNR to linear scale
    Esym=np.sum(np.power(np.abs(x),2))/L; #Calculate actual symbol energy
    N0=Esym/SNR; #Find the noise spectral density
    if(np.isreal(x).all()):
        noiseSigma = np.sqrt(N0);#Standard deviation for AWGN Noise when x is real
        n = noiseSigma*np.random.normal(0,1,L);#computed noise
    else:
        noiseSigma=np.sqrt(N0/2);#Standard deviation for AWGN Noise when x is complex
        n = noiseSigma*(np.random.normal(0,1,L)+1j*np.random.normal(0,1,L));#computed noise
    y = x + n; #received signal
    return y

def bin2bipolar(vec):
    temp=np.copy(vec)
    for i in range(len(vec)):
        if temp[i]==0:
            temp[i]=-1
    return vec


def binary_counting_matrix(n):
    #creation of binary table
    B = [i for i in product(range(2), repeat=n)]

    #transpose of the matrix B
    Bt=np.transpose(B)

    #dot product of the matrix B and her traspose
    Hn=B@Bt

    #module 2 on the matrix Hn
    for i in range (len(Hn)):
        for j in range (len(Hn)):
            Hn[i][j]=Hn[i][j]%2

    #switching every 0 -> 1 and 1 -> -1
    for i in range (len(Hn)):
        for j in range (len(Hn)):
            if(Hn[i][j]==0):
                Hn[i][j]=1
            elif(Hn[i][j]==1):
                Hn[i][j]=-1
            else:
                print("error")
    return Hn

##################################classss#########


class mainPage:
    global path0,path1,img0,img1,user0mat,user1mat,sendData
    ser = 0

    def load_img0(self):
        self.path0 = tk.filedialog.askopenfilename(
            title='Open Image File')
        print('Selected path: %s' % self.path0)
        self.img0 = ImageTk.PhotoImage(Image.open(self.path0))
        self.user0mat = ndimage.imread(self.path0, mode='L')

    def load_img1(self):
        self.path1 = tk.filedialog.askopenfilename(
            title='Open Image File')
        print('Selected path: %s' % self.path1)
        self.img1 = ImageTk.PhotoImage(Image.open(self.path1))
        self.user1mat = ndimage.imread(self.path1, mode='L')

    def show_img(self):
        self.img0C.create_image((0, 0), anchor='nw', image=self.img0)
        #self.img0C.pack()
        self.img1C.create_image((0, 0), anchor='nw', image=self.img1)
        # self.img1C.pack()

    def init_serial(self):
        u1=np.array(list(bytes(self.entry1.get(),'ascii')))
        u2=np.array(list(bytes(self.entry2.get(),'ascii')))
        user0vec=np.array([])
        user1vec=np.array([])
        for i in u1:
            user0vec = np.append(user0vec, bitfield(i)).astype(int)
        for i in u2:
            user1vec = np.append(user1vec, bitfield(i)).astype(int)


        if(len(user0vec)>len(user1vec)):
            user1vec=np.copy(np.pad(user1vec,(0,len(user0vec)-len(user1vec)),'constant',constant_values=(0)))
        else:
            user0vec=np.copy(np.pad(user0vec,(0,len(user1vec)-len(user0vec)),'constant',constant_values=(0)))

        bi_user0vec = np.copy(bin2bipolar(user0vec))
        bi_user1vec = np.copy(bin2bipolar(user1vec))

        # encode data
        dataMat = np.array([bi_user0vec, bi_user1vec])
        w = binary_counting_matrix(1)
        self.sendData = encoder_data(dataMat, w)
        print("done encoding")


    def encode_b(self):
        start_time1=time.time()
        if( len(self.entry1.get())>0 and len(self.entry2.get())>0 ):
            {
                #user0vec=np.copy(self.entry1.get())

            }
        user0vec = convert_img2vector(self.user0mat)
        user1vec = convert_img2vector(self.user1mat)
        bi_user0vec = np.copy(bin2bipolar(user0vec))
        bi_user1vec = np.copy(bin2bipolar(user1vec))

        # encode data
        dataMat = np.array([bi_user0vec, bi_user1vec])
        w = binary_counting_matrix(1)
        self.sendData = encoder_data(dataMat, w)
        print("done encoding, it took: ", time.time()-start_time1)

    def send_data(self):
        #B8:27:EB:FE:23:00
        #B8:27:EB:6D:B1:44

        self.client = bluetooth.BluetoothSocket(bluetooth.RFCOMM)
        self.client.connect(('B8:27:EB:6D:B1:44', 1))
        ######

        ####
        print("1")
        sent_AWGN = np.array(add_awgn_noise(self.sendData, int(self.SNR_db.get())))
        start_time1=time.time()
        raw_sent_data = sent_AWGN.tobytes()
        self.client.send(int(len(raw_sent_data)).to_bytes(4, byteorder='little'))
        sent = 0
        print("2")
        self.client.send(raw_sent_data)
        self.client.recv(4)
        self.client.close()

        # ####
        # self.client1 = bluetooth.BluetoothSocket(bluetooth.RFCOMM)
        # self.client1.connect(('B8:27:EB:FE:23:00', 1))
        # self.client1.send(int(len(raw_sent_data)).to_bytes(4, byteorder='little'))
        # self.client1.send(raw_sent_data)
        # self.client1.recv(4)
        # self.client1.close()


    ###
        #self.ser.write(raw_sent_data)
        # self.ser.write(test_str.encode('utf-8'))
        #print(raw_sent_data)
        print("wrote successfully, it took: ", time.time()-start_time1)

    def __init__(self, top=None):
        '''This class configures and populates the toplevel window.
           top is the toplevel containing window.'''
        _bgcolor = '#d9d9d9'  # X11 color: 'gray85'
        _fgcolor = '#000000'  # X11 color: 'black'
        _compcolor = '#d9d9d9' # X11 color: 'gray85'
        _ana1color = '#d9d9d9' # X11 color: 'gray85'
        _ana2color = '#d9d9d9' # X11 color: 'gray85'
        self.style = ttk.Style()
        if sys.platform == "win32":
            self.style.theme_use('winnative')
        self.style.configure('.',background=_bgcolor)
        self.style.configure('.',foreground=_fgcolor)
        self.style.configure('.',font="TkDefaultFont")
        self.style.map('.',background=
            [('selected', _compcolor), ('active',_ana2color)])

        top.geometry("437x255+564+348")
        top.title("CDMA Transmitter ")
        top.configure(background="#d9d9d9")

        self.openB = tk.Button(top)
        self.openB.place(relx=0.5, rely=0.5, height=24, width=81)
        self.openB.configure(activebackground="#d9d9d9")
        self.openB.configure(activeforeground="#000000")
        self.openB.configure(background="#d9d9d9")
        self.openB.configure(disabledforeground="#a3a3a3")
        self.openB.configure(foreground="#000000")
        self.openB.configure(highlightbackground="#d9d9d9")
        self.openB.configure(highlightcolor="black")
        self.openB.configure(pady="0")
        self.openB.configure(text='''testing button ''')
        self.openB.configure(command=self.init_serial)


        # self.ports=tk.StringVar()
        # self.portOpt = ttk.Combobox(top, textvariable=self.ports)
        # self.portOpt['values']=(0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        # self.portOpt.place(relx=0.046, rely=0.078, relheight=0.094
        #         , relwidth=0.185)
        # self.portOpt.configure(width=83)
        # self.portOpt.configure(takefocus="")
        # self.portOpt.current(4)

        self.loadB0 = tk.Button(top)
        self.loadB0.place(relx=0.046, rely=0.078, height=24, width=81)
        self.loadB0.configure(activebackground="#d9d9d9")
        self.loadB0.configure(activeforeground="#000000")
        self.loadB0.configure(background="#d9d9d9")
        self.loadB0.configure(disabledforeground="#a3a3a3")
        self.loadB0.configure(foreground="#000000")
        self.loadB0.configure(highlightbackground="#d9d9d9")
        self.loadB0.configure(highlightcolor="black")
        self.loadB0.configure(pady="0")
        self.loadB0.configure(text='''Load img 0''')
        self.loadB0.configure(command=self.load_img0)

        self.loadB1 = tk.Button(top)
        self.loadB1.place(relx=0.275, rely=0.078, height=24, width=81)
        self.loadB1.configure(activebackground="#d9d9d9")
        self.loadB1.configure(activeforeground="#000000")
        self.loadB1.configure(background="#d9d9d9")
        self.loadB1.configure(disabledforeground="#a3a3a3")
        self.loadB1.configure(foreground="#000000")
        self.loadB1.configure(highlightbackground="#d9d9d9")
        self.loadB1.configure(highlightcolor="black")
        self.loadB1.configure(pady="0")
        self.loadB1.configure(text='''Load img 1''')
        self.loadB1.configure(command=self.load_img1)

        self.showB = tk.Button(top)
        self.showB.place(relx=0.504, rely=0.078, height=24, width=81)
        self.showB.configure(activebackground="#d9d9d9")
        self.showB.configure(activeforeground="#000000")
        self.showB.configure(background="#d9d9d9")
        self.showB.configure(disabledforeground="#a3a3a3")
        self.showB.configure(foreground="#000000")
        self.showB.configure(highlightbackground="#d9d9d9")
        self.showB.configure(highlightcolor="black")
        self.showB.configure(pady="0")
        self.showB.configure(text='''Show images''')
        self.showB.configure(command=self.show_img)

        self.encodeB = tk.Button(top)
        self.encodeB.place(relx=0.046, rely=0.549, height=24, width=81)
        self.encodeB.configure(activebackground="#d9d9d9")
        self.encodeB.configure(activeforeground="#000000")
        self.encodeB.configure(background="#d9d9d9")
        self.encodeB.configure(disabledforeground="#a3a3a3")
        self.encodeB.configure(foreground="#000000")
        self.encodeB.configure(highlightbackground="#d9d9d9")
        self.encodeB.configure(highlightcolor="black")
        self.encodeB.configure(pady="0")
        self.encodeB.configure(text='''Encode data''')
        self.encodeB.configure(command=self.encode_b)

        self.lable1=tk.Label(top)
        self.lable1.place(relx=0.046, rely=0.235, height=24, width=81)
        self.lable1.configure(activebackground="#d9d9d9")
        self.lable1.configure(activeforeground="#000000")
        self.lable1.configure(background="#d9d9d9")
        self.lable1.configure(disabledforeground="#a3a3a3")
        self.lable1.configure(foreground="#000000")
        self.lable1.configure(highlightbackground="#d9d9d9")
        self.lable1.configure(highlightcolor="black")
        self.lable1.configure(pady="0")
        self.lable1.configure(text='''Text for user 1''')
        self.lable2=tk.Label(top)

        self.entry1=tk.Entry(top)
        self.entry1.place(relx=0.275, rely=0.235, height=24, width=81)
        self.entry1.configure(disabledforeground="#a3a3a3")
        self.entry1.configure(foreground="#000000")
        self.entry1.configure(highlightbackground="#d9d9d9")
        self.entry1.configure(highlightcolor="black")

        self.lable2.place(relx=0.046, rely=0.392, height=24, width=81)
        self.lable2.configure(activebackground="#d9d9d9")
        self.lable2.configure(activeforeground="#000000")
        self.lable2.configure(background="#d9d9d9")
        self.lable2.configure(disabledforeground="#a3a3a3")
        self.lable2.configure(foreground="#000000")
        self.lable2.configure(highlightbackground="#d9d9d9")
        self.lable2.configure(highlightcolor="black")
        self.lable2.configure(pady="0")
        self.lable2.configure(text='''Text for user 2''')

        self.entry2=tk.Entry(top)
        self.entry2.place(relx=0.275, rely=0.392, height=24, width=81)
        self.entry2.configure(disabledforeground="#a3a3a3")
        self.entry2.configure(foreground="#000000")
        self.entry2.configure(highlightbackground="#d9d9d9")
        self.entry2.configure(highlightcolor="black")


        self.SendB = tk.Button(top)
        self.SendB.place(relx=0.275, rely=0.706, height=24, width=81)
        self.SendB.configure(activebackground="#d9d9d9")
        self.SendB.configure(activeforeground="#000000")
        self.SendB.configure(background="#d9d9d9")
        self.SendB.configure(disabledforeground="#a3a3a3")
        self.SendB.configure(foreground="#000000")
        self.SendB.configure(highlightbackground="#d9d9d9")
        self.SendB.configure(highlightcolor="black")
        self.SendB.configure(pady="0")
        self.SendB.configure(text='''Send''')
        self.SendB.configure(command=self.send_data)


        self.SNR_db=tk.StringVar()

        self.SNR_opt = ttk.Combobox(top)
        self.SNR_opt.place(relx=0.046, rely=0.706, relheight=0.082
                , relwidth=0.185)
        self.SNR_opt['values']=(5,6,7,8,9,10,11,12,13,14,15)
        self.SNR_opt.configure(textvariable=self.SNR_db)
        self.SNR_opt.configure(width=63)
        self.SNR_opt.configure(takefocus="")
        self.SNR_opt.current(9)

        self.img0C = tk.Canvas(top)
        self.img0C.place(relx=0.618, rely=0.314, relheight=0.196, relwidth=0.114)

        self.img0C.configure(background="#d9d9d9")
        self.img0C.configure(borderwidth="2")
        self.img0C.configure(insertbackground="black")
        self.img0C.configure(relief='ridge')
        self.img0C.configure(selectbackground="#c4c4c4")
        self.img0C.configure(selectforeground="black")
        self.img0C.configure(width=125)

        self.img1C = tk.Canvas(top)
        self.img1C.place(relx=0.801, rely=0.314, relheight=0.196, relwidth=0.114)

        self.img1C.configure(background="#d9d9d9")
        self.img1C.configure(borderwidth="2")
        self.img1C.configure(insertbackground="black")
        self.img1C.configure(relief='ridge')
        self.img1C.configure(selectbackground="#c4c4c4")
        self.img1C.configure(selectforeground="black")
        self.img1C.configure(width=125)



if __name__ == '__main__':
    vp_start_gui()





