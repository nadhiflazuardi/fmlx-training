## What is serial communication?
---
Imagine sending a message through a single straight line, bit per bit, in order.

It's the opposite of parallel communication where multiple parts can be sent at the same time.


## What is UART?
---
UART stands for Universal Asynchronous Receiver/Transmitter.

Think of a pair of package expedition worker whose job is to breakdown your "package" into chunks before being sent, and re-assemble it on the other end.

Technically, it's a hardware module that turns data into serial bit stream.


## Side Note
---
- Disable shell via serial in `raspi-config` to make sure program runs properly

# Making a Simple Receiver and Transmitter 
### Receiver
```csharp
using System;
using System.IO.Ports;

namespace MySerialApp
{
    class Program
    {
        static void Main(string[] args)
        {
            // Replace with your desired port name (e.g., "/dev/serial0")
            string portName = "/dev/serial0";
            int baudRate = 9600;

            using (SerialPort serialPort = new SerialPort(portName, baudRate))
            {
                try
                {
                    serialPort.Open();
                    Console.WriteLine($"Serial port {portName} opened successfully.");
                    Console.WriteLine("Listening for messages... Press Ctrl+C to exit.");

                    Console.CancelKeyPress += delegate
                    {
                        serialPort.Close();
                        Console.WriteLine("Serial Port closed");
                    };

                    while (true)
                    {
                        try
                        {
                            // Read incoming data
                            string receivedData = serialPort.ReadLine();
                            Console.WriteLine($"Received: {receivedData}");
                        }
                        catch (TimeoutException)
                        {
                            // ReadLine timeout - continue listening
                            continue;
                        }
                    }
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Error: {ex.Message}");
                }
            }
        }
    }
}

```

### Transmitter
```csharp
using System;
using System.IO.Ports;

namespace MySerialApp
{
    class Program
    {
        static void Main(string[] args)
        {
            // Replace with your desired port name (e.g., "/dev/serial0")
            string portName = "/dev/serial0";
            int baudRate = 9600;

            using (SerialPort serialPort = new SerialPort(portName, baudRate))
            {
                try
                {
                    serialPort.Open();
                    Console.WriteLine($"Serial port {portName} opened successfully.");

                    Console.CancelKeyPress += delegate
                    {
                        serialPort.Close();
                        Console.WriteLine("Serial Port closed");
                    };

                    int count = 0;
                    while (true)
                    {
                        serialPort.WriteLine("Hello from .NET! Message count = " + count);
                        count += 1;
                    }
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Error: {ex.Message}");
                }
            }
        }
    }
}

```

# Making a Simple Chat App using Serial UART
```csharp
using System;
using System.IO.Ports;
using System.Threading;

namespace SerialChat
{
  class Program
  {
    static void Main(string[] args)
    {
      const string portName = "/dev/serial0";
      const int baudRate = 9600;

      SerialPort serialPort = new SerialPort(portName, baudRate)
      {
        Parity = Parity.None,
        DataBits = 8,
        StopBits = StopBits.One,
        Handshake = Handshake.None,
        ReadTimeout = 500,
        WriteTimeout = 500
      };

      try
      {
        serialPort.Open();
        Console.WriteLine($"Connected to {portName} at {baudRate} bps.");
        Console.WriteLine("Type messages and press Enter to send.\n");

        // Thread for receiving messages
        Thread receiveThread = new Thread(() =>
        {
          while (true)
          {
            try
            {
              string incoming = serialPort.ReadLine();
              Console.ForegroundColor = ConsoleColor.Green;
              Console.WriteLine($"\nFriend: {incoming}");
              Console.ResetColor();
              Console.Write("You: ");
            }
            catch (TimeoutException)
            {
              // Ignore read timeouts
            }
            catch (Exception ex)
            {
              Console.WriteLine($"[Receive Error] {ex.Message}");
              break;
            }
          }
        });

        receiveThread.IsBackground = true;
        receiveThread.Start();

        // Sending loop
        while (true)
        {
          Console.Write("You: ");
          string message = Console.ReadLine();
          if (string.IsNullOrWhiteSpace(message)) continue;

          try
          {
            serialPort.WriteLine(message);
          }
          catch (Exception ex)
          {
            Console.WriteLine($"[Send Error] {ex.Message}");
            break;
          }
        }
      }
      catch (Exception ex)
      {
        Console.WriteLine($"[Connection Error] {ex.Message}");
      }
      finally
      {
        if (serialPort.IsOpen)
          serialPort.Close();
      }
    }
  }
}

```

# Virtual Serial
---
### Prerequisites
- Install socat
- Install putty

### Create Virtual COM Ports
```bash
   sudo socat PTY,link=/dev/ttyV0,raw,echo=0 PTY,link=/dev/ttyV1,raw,echo=0
```
- **PTY** creates pseudo-terminal (virtual serial port)
- **link** specify the paths to the virtual ports
- **raw** disables line discipline, characters will be sent as-is
- **echo=0** to disable local echoing, avoiding duplicate characters

Verify virtual ports are created
`ls /dev/ttyV*`

### Simulate Serial Communication
Open 2 PuTTY windows

Open serial connection
- Select Serial for the connection type
- Enter correct path to the virtual port
- (optional) configure other settings as needed

## Test Communication
In one window, try typing something

The text should be visible from the other window