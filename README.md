# nanta
codigo 
import tkinter as tk
import serial
import struct
import csv
import datetime

class App:
    def __init__(self, master):
        self.master = master
        self.running = False
        self.slave_id = 1
        self.function_code = 3
        self.starting_address = 11
        self.number_of_registers = 1
        self.crc = 0xFFFF
        #self.master.iconbitmap("/home/rock/Downloads/1.ico") 
        self.ser1 = serial.Serial('/dev/ttyUSB0', 9600, timeout=0.2)
        self.ser2 = serial.Serial('/dev/ttyUSB1', 9600, timeout=0.2)

        self.frame = tk.Frame(self.master, bg="lightblue")
        self.frame.pack()

        self.start_button = tk.Button(self.frame, text='Start', command=self.start_measurement, bg="green", height=3, width=15,relief="raised",bd=8, highlightthickness=3)
        self.start_button.grid(row=0, column=0, padx=15, pady=100)
        self.start_button.config(activebackground="yellow")

        self.stop_button = tk.Button(self.frame, text='Stop', command=self.stop_measurement, bg="red", height=3, width=15,relief="raised",bd=8, highlightthickness=3)
        self.stop_button.grid(row=0, column=1, padx=15, pady=100)
        self.stop_button.config(activebackground="orange")

        self.data_label = tk.Label(self.master, text='Sensor Data Granuladora:',bg="lightyellow",relief="sunken",bd=2,)
        self.data_label.pack()

        self.master.geometry("600x500".format(self.master.winfo_width()*4, self.master.winfo_height()*4))

    def start_measurement(self):
        self.running = True
        self.start_button.config(bg="yellow", text="Start (pressed)")

    def stop_measurement(self):
        self.running = False
        self.stop_button.config(bg="orange", text="Stop (pressed)")
        
    def start_measurement(self):
        self.running = True
        self.write_csv_header()
        self.read_sensor_data()

    def stop_measurement(self):
        self.running = False

    def read_sensor_data(self):
        if not self.running:
            return

        request = struct.pack('>BBHH', self.slave_id, self.function_code, self.starting_address, self.number_of_registers)
        crc = self.crc & 0xFFFF
        for b in request:
            crc = crc ^ b
            for i in range(8):
                if crc & 0x0001:
                    crc = (crc >> 1) ^ 0xA001
                else:
                    crc = crc >> 1
        request += struct.pack('<H', crc)

        self.ser1.write(request)
        response1 = self.ser1.read(5 + 2 * self.number_of_registers)

        self.ser2.write(request)
        response2 = self.ser2.read(5 + 2 * self.number_of_registers)

        if response1[1] != self.function_code or response2[1] != self.function_code:
            self.data_label.config(text='Error: incorrect function code')
        elif response1[2] != 2 * self.number_of_registers or response2[2] != 2 * self.number_of_registers:
            self.data_label.config(text='Error: incorrect number of bytes')
        else:
            registers1 = struct.unpack('>' + 'H'*self.number_of_registers, response1[3:-2])
            registers2 = struct.unpack('>' + 'H'*self.number_of_registers, response2[3:-2])
            self.data_label.config(text='Sensor 1 Data: {}  Sensor 2 Data: {}'.format(registers1, registers2))
            self.write_csv_data(registers1, registers2)

        self.master.after(1000, self.read_sensor_data)

    def write_csv_header(self):
        fields = ['Datetime', 'Sensor 1 Data', 'Sensor 2 Data']
        with open('data.csv', 'a', newline='') as csvfile:
            writer = csv.writer(csvfile)
            writer.writerow(fields)

    def write_csv_data(self, registers1, registers2):
        fields = [datetime.datetime.now().strftime('%Y-%m-%d %H:%M:%S'), registers1[0], registers2[0]]
        with open('data.csv', 'a', newline='') as csvfile:
            writer = csv.writer(csvfile)
            writer.writerow(fields)

root = tk.Tk()
app = App(root)
root.mainloop()
