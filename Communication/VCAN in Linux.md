## Setup VCAN Interface
```bash
# Load vcan kernel module
sudo modprobe vcan

# Create virtual CAN interface
sudo ip link add dev vcan0 type vcan

# Bring interface up
sudo ip link set up vcan0

# Verify interface is up
ip link show vcan0
```


## Example of Automobile System
### EngineECU
```csharp
using System;
using System.Threading.Tasks;
using System.Net.Sockets;
using System.Runtime.InteropServices;
using SocketCANSharp;

namespace EngineECU
{
  class Program
  {
    private static bool _running = true;
    private static SafeFileDescriptorHandle _socketHandle;

    // Engine parameters
    private static int _engineRpm = 800;        // RPM
    private static int _engineTemp = 85;        // Celsius
    private static bool _engineRunning = true;  // Engine status
    private static int _throttlePosition = 0;   // 0-100%

    static async Task Main(string[] args)
    {
      Console.WriteLine("=== ENGINE ECU SIMULATOR ===");
      Console.WriteLine("Sending: RPM, Temperature, Status, Throttle");
      Console.WriteLine("Listening: Brake signals, Diagnostic requests");
      Console.WriteLine();

      try
      {
        await InitializeCAN();

        Console.WriteLine("Commands:");
        Console.WriteLine("  '+' / '-' : Increase/Decrease RPM");
        Console.WriteLine("  'h' / 'c' : Heat up/Cool down engine");
        Console.WriteLine("  't' / 'r' : Increase/Decrease throttle");
        Console.WriteLine("  'e'       : Toggle engine on/off");
        Console.WriteLine("  'q'       : Quit");
        Console.WriteLine();

        // Start background tasks
        var sendTask = Task.Run(SendEngineData);
        var listenTask = Task.Run(ListenToCANBus);
        var inputTask = Task.Run(HandleUserInput);

        await Task.WhenAny(inputTask);
        _running = false;

        await Task.WhenAll(sendTask, listenTask);
      }
      catch (Exception ex)
      {
        Console.WriteLine($"Error: {ex.Message}");
      }
      finally
      {
        _socketHandle?.Dispose();
      }
    }

    static async Task InitializeCAN()
    {
      _socketHandle = LibcNativeMethods.Socket(
        SocketCanConstants.PF_CAN,
        SocketType.Raw,
        SocketCanProtocolType.CAN_RAW
      );

      if (_socketHandle.IsInvalid)
        throw new Exception("Failed to create CAN socket");

      var ifr = new Ifreq("vcan0");
      int ioctlResult = LibcNativeMethods.Ioctl(_socketHandle, SocketCanConstants.SIOCGIFINDEX, ifr);
      if (ioctlResult == -1)
        throw new Exception("vcan0 interface not found. Run: sudo modprobe vcan && sudo ip link add dev vcan0 type vcan && sudo ip link set up vcan0");

      var addr = new SockAddrCan(ifr.IfIndex);
      int bindResult = LibcNativeMethods.Bind(_socketHandle, addr, Marshal.SizeOf(typeof(SockAddrCan)));
      if (bindResult == -1)
        throw new Exception("Failed to bind to vcan0");

      Console.WriteLine("✓ Connected to CAN bus (vcan0) as ENGINE ECU");
    }

    static async Task SendEngineData()
    {
      while (_running)
      {
        try
        {
          // Send RPM every 50ms (20Hz) - Critical for other ECUs
          SendRPM();
          await Task.Delay(5000);

          // Send temperature every 1000ms (1Hz) - Less critical
          if (DateTime.Now.Millisecond < 50)
            SendTemperature();

          // Send throttle position every 100ms (10Hz)
          if (DateTime.Now.Millisecond % 100 < 50)
            SendThrottlePosition();

          // Send engine status every 500ms (2Hz)
          if (DateTime.Now.Millisecond % 500 < 50)
            SendEngineStatus();
        }
        catch (Exception ex)
        {
          Console.WriteLine($"Send error: {ex.Message}");
          await Task.Delay(10000);
        }
      }
    }

    static void SendRPM()
    {
      var frame = new CanFrame
      {
        CanId = 0x110,
        Length = 2,
        Data = new byte[8]
      };

      frame.Data[0] = (byte)(_engineRpm & 0xFF);
      frame.Data[1] = (byte)((_engineRpm >> 8) & 0xFF);

      SendCANFrame(frame, $"ENGINE RPM: {_engineRpm}");
    }

    static void SendTemperature()
    {
      var frame = new CanFrame
      {
        CanId = 0x111,
        Length = 1,
        Data = new byte[8]
      };

      frame.Data[0] = (byte)_engineTemp;
      SendCANFrame(frame, $"ENGINE TEMP: {_engineTemp}°C");
    }

    static void SendThrottlePosition()
    {
      var frame = new CanFrame
      {
        CanId = 0x112,
        Length = 1,
        Data = new byte[8]
      };

      frame.Data[0] = (byte)_throttlePosition;
      SendCANFrame(frame, $"THROTTLE: {_throttlePosition}%");
    }

    static void SendEngineStatus()
    {
      var frame = new CanFrame
      {
        CanId = 0x113,
        Length = 1,
        Data = new byte[8]
      };

      frame.Data[0] = (byte)(_engineRunning ? 1 : 0);
      SendCANFrame(frame, $"ENGINE: {(_engineRunning ? "ON" : "OFF")}");
    }

    static void SendCANFrame(CanFrame frame, string description)
    {
      try
      {
        int frameSize = Marshal.SizeOf(typeof(CanFrame));
        int bytesWritten = LibcNativeMethods.Write(_socketHandle, ref frame, frameSize);

        if (bytesWritten > 0)
        {
          Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] TX → 0x{frame.CanId:X3}: {description}");
        }
      }
      catch (Exception ex)
      {
        Console.WriteLine($"Failed to send {description}: {ex.Message}");
      }
    }

    static async Task ListenToCANBus()
    {
      int frameSize = Marshal.SizeOf(typeof(CanFrame));

      while (_running)
      {
        try
        {
          var frame = new CanFrame();
          int bytesRead = LibcNativeMethods.Read(_socketHandle, ref frame, frameSize);

          if (bytesRead > 0)
          {
            ProcessIncomingMessage(frame);
          }
        }
        catch (Exception ex)
        {
          if (_running)
            Console.WriteLine($"Listen error: {ex.Message}");
        }

        await Task.Delay(1000);
      }
    }

    static void ProcessIncomingMessage(CanFrame frame)
    {
      string timestamp = DateTime.Now.ToString("HH:mm:ss.fff");

      switch (frame.CanId)
      {
        case 0x220:
          bool brakesApplied = frame.Data[0] == 1;
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: BRAKE {(brakesApplied ? "APPLIED" : "RELEASED")}");
          if (brakesApplied && _engineRunning && _engineRpm > 800)
          {
            _engineRpm = Math.Max(800, _engineRpm - 300);
            Console.WriteLine($"            → Engine RPM reduced to {_engineRpm} due to braking");
          }
          break;

        case 0x330:
          int gear = frame.Data[0];
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: GEAR CHANGED TO {gear}");
          break;

        case 0x7DF:
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: DIAGNOSTIC REQUEST");
          break;

        default:
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: Unknown message");
          break;
      }
    }

    static async Task HandleUserInput()
    {
      while (_running)
      {
        var key = Console.ReadKey(true);

        switch (key.KeyChar)
        {
          case '+':
            if (_engineRunning && _engineRpm < 6000)
            {
              _engineRpm = Math.Min(6000, _engineRpm + 200);
              Console.WriteLine($"RPM increased to: {_engineRpm}");
            }
            break;

          case '-':
            if (_engineRunning && _engineRpm > 600)
            {
              _engineRpm = Math.Max(600, _engineRpm - 200);
              Console.WriteLine($"RPM decreased to: {_engineRpm}");
            }
            break;

          case 'h':
            _engineTemp = Math.Min(120, _engineTemp + 10);
            Console.WriteLine($"Engine temp: {_engineTemp}°C");
            break;

          case 'c':
            _engineTemp = Math.Max(40, _engineTemp - 10);
            Console.WriteLine($"Engine temp: {_engineTemp}°C");
            break;

          case 't':
            _throttlePosition = Math.Min(100, _throttlePosition + 10);
            Console.WriteLine($"Throttle: {_throttlePosition}%");
            break;

          case 'r':
            _throttlePosition = Math.Max(0, _throttlePosition - 10);
            Console.WriteLine($"Throttle: {_throttlePosition}%");
            break;

          case 'e':
            _engineRunning = !_engineRunning;
            _engineRpm = _engineRunning ? 800 : 0;
            _throttlePosition = _engineRunning ? 0 : 0;
            Console.WriteLine($"Engine: {(_engineRunning ? "STARTED" : "STOPPED")}");
            break;

          case 'q':
            _running = false;
            Console.WriteLine("Shutting down Engine ECU...");
            return;
        }
      }
    }
  }
}

```

### DashboardECU
```csharp
using System;
using System.Threading.Tasks;
using System.Net.Sockets;
using System.Runtime.InteropServices;
using SocketCANSharp;

namespace DashboardECU
{
  class Program
  {
    private static bool _running = true;
    private static SafeFileDescriptorHandle _socketHandle;

    // Dashboard display data (collected from other ECUs)
    private static int _displayRpm = 0;
    private static int _displaySpeed = 0;
    private static int _displayGear = 1;
    private static int _engineTemp = 0;
    private static int _fuelLevel = 75; // Percentage
    private static bool _engineRunning = false;
    private static bool _brakesApplied = false;
    private static bool _absActive = false;

    // Warning lights
    private static bool _checkEngineLight = false;
    private static bool _overheatingWarning = false;
    private static bool _lowFuelWarning = false;

    static async Task Main(string[] args)
    {
      Console.WriteLine("=== DASHBOARD ECU SIMULATOR ===");
      Console.WriteLine("Sending: Fuel Level, Warning Lights");
      Console.WriteLine("Listening: ALL other ECUs for display");
      Console.WriteLine();

      try
      {
        await InitializeCAN();

        Console.WriteLine("Commands:");
        Console.WriteLine("  'f' / 'F' : Decrease/Increase fuel level");
        Console.WriteLine("  'w'       : Toggle check engine light");
        Console.WriteLine("  'd'       : Show current dashboard display");
        Console.WriteLine("  'q'       : Quit");
        Console.WriteLine();

        var sendTask = Task.Run(SendDashboardData);
        var listenTask = Task.Run(ListenToCANBus);
        var displayTask = Task.Run(UpdateDashboardDisplay);
        var inputTask = Task.Run(HandleUserInput);

        await Task.WhenAny(inputTask);
        _running = false;

        await Task.WhenAll(sendTask, listenTask, displayTask);
      }
      catch (Exception ex)
      {
        Console.WriteLine($"Error: {ex.Message}");
      }
      finally
      {
        _socketHandle?.Dispose();
      }
    }

    static async Task InitializeCAN()
    {
      _socketHandle = LibcNativeMethods.Socket(SocketCanConstants.PF_CAN, SocketType.Raw, SocketCanProtocolType.CAN_RAW);
      if (_socketHandle.IsInvalid)
        throw new Exception("Failed to create CAN socket");

      var ifr = new Ifreq("vcan0");
      int ioctlResult = LibcNativeMethods.Ioctl(_socketHandle, SocketCanConstants.SIOCGIFINDEX, ifr);
      if (ioctlResult == -1)
        throw new Exception("vcan0 interface not found");

      var addr = new SockAddrCan(ifr.IfIndex);
      int bindResult = LibcNativeMethods.Bind(_socketHandle, addr, Marshal.SizeOf(typeof(SockAddrCan)));
      if (bindResult == -1)
        throw new Exception("Failed to bind to vcan0");

      Console.WriteLine("✓ Connected to CAN bus (vcan0) as DASHBOARD ECU");
    }

    static async Task SendDashboardData()
    {
      while (_running)
      {
        try
        {
          if (DateTime.Now.Millisecond < 100)
            SendFuelLevel();

          if (DateTime.Now.Millisecond % 1000 < 100)
            SendWarningLights();

          await Task.Delay(10000);
        }
        catch (Exception ex)
        {
          Console.WriteLine($"Send error: {ex.Message}");
          await Task.Delay(10000);
        }
      }
    }

    static void SendFuelLevel()
    {
      var frame = new CanFrame
      {
        CanId = 0x440,
        Length = 1,
        Data = new byte[8]
      };

      frame.Data[0] = (byte)_fuelLevel;
      SendCANFrame(frame, $"FUEL LEVEL: {_fuelLevel}%");
    }

    static void SendWarningLights()
    {
      var frame = new CanFrame
      {
        CanId = 0x441,
        Length = 3,
        Data = new byte[8]
      };

      frame.Data[0] = (byte)(_checkEngineLight ? 1 : 0);
      frame.Data[1] = (byte)(_overheatingWarning ? 1 : 0);
      frame.Data[2] = (byte)(_lowFuelWarning ? 1 : 0);

      string warnings = "";
      if (_checkEngineLight) warnings += "CHECK_ENGINE ";
      if (_overheatingWarning) warnings += "OVERHEAT ";
      if (_lowFuelWarning) warnings += "LOW_FUEL ";
      if (string.IsNullOrEmpty(warnings)) warnings = "NONE";

      SendCANFrame(frame, $"WARNINGS: {warnings.Trim()}");
    }

    static void SendCANFrame(CanFrame frame, string description)
    {
      try
      {
        int frameSize = Marshal.SizeOf(typeof(CanFrame));
        int bytesWritten = LibcNativeMethods.Write(_socketHandle, ref frame, frameSize);

        if (bytesWritten > 0)
        {
          Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] TX → 0x{frame.CanId:X3}: {description}");
        }
      }
      catch (Exception ex)
      {
        Console.WriteLine($"Failed to send {description}: {ex.Message}");
      }
    }

    static async Task ListenToCANBus()
    {
      int frameSize = Marshal.SizeOf(typeof(CanFrame));

      while (_running)
      {
        try
        {
          var frame = new CanFrame();
          int bytesRead = LibcNativeMethods.Read(_socketHandle, ref frame, frameSize);

          if (bytesRead > 0)
          {
            ProcessIncomingMessage(frame);
          }
        }
        catch (Exception ex)
        {
          if (_running)
            Console.WriteLine($"Listen error: {ex.Message}");
        }

        await Task.Delay(1000);
      }
    }

    static void ProcessIncomingMessage(CanFrame frame)
    {
      string timestamp = DateTime.Now.ToString("HH:mm:ss.fff");

      switch (frame.CanId)
      {
        case 0x110:
          _displayRpm = frame.Data[0] | (frame.Data[1] << 8);
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: ENGINE RPM {_displayRpm}");
          break;

        case 0x111:
          _engineTemp = frame.Data[0];
          _overheatingWarning = _engineTemp > 100;
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: ENGINE TEMP {_engineTemp}°C");
          break;

        case 0x113:
          _engineRunning = frame.Data[0] == 1;
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: ENGINE {(_engineRunning ? "ON" : "OFF")}");
          break;

        case 0x220:
          _displaySpeed = frame.Data[0] | (frame.Data[1] << 8);
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: SPEED {_displaySpeed} km/h");
          break;

        case 0x221:
          _brakesApplied = frame.Data[0] == 1;
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: BRAKE {(_brakesApplied ? "ON" : "OFF")}");
          break;

        case 0x222:
          _absActive = frame.Data[0] == 1;
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: ABS {(_absActive ? "ACTIVE" : "INACTIVE")}");
          break;

        case 0x330:
          _displayGear = frame.Data[0];
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: GEAR {_displayGear}");
          break;

        case 0x331:
          int transTemp = frame.Data[0];
          Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: TRANS TEMP {transTemp}°C");
          break;

        default:
          break;
      }
    }

    static async Task UpdateDashboardDisplay()
    {
      while (_running)
      {
        try
        {
          if (_engineRunning && _displayRpm > 800)
          {
            double fuelConsumptionRate = (_displayRpm / 6000.0) * 0.01;
            _fuelLevel = Math.Max(0, (int)(_fuelLevel - fuelConsumptionRate));
            _lowFuelWarning = _fuelLevel < 15;
          }

          await Task.Delay(10000);
        }
        catch (Exception ex)
        {
          Console.WriteLine($"Display update error: {ex.Message}");
        }
      }
    }

    static void ShowDashboard()
    {
      Console.WriteLine("\n╔════════════════════════════════════╗");
      Console.WriteLine("║           DASHBOARD DISPLAY           ║");
      Console.WriteLine("╠════════════════════════════════════╣");
      Console.WriteLine($"║ RPM:     {_displayRpm,6} rpm             ║");
      Console.WriteLine($"║ Speed:   {_displaySpeed,6} km/h             ║");
      Console.WriteLine($"║ Gear:    {_displayGear,6}                 ║");
      Console.WriteLine($"║ Engine:  {(_engineRunning ? "RUNNING" : "STOPPED"),7}             ║");
      Console.WriteLine($"║ Temp:    {_engineTemp,6}°C               ║");
      Console.WriteLine($"║ Fuel:    {_fuelLevel,6}%                ║");
      Console.WriteLine("╠════════════════════════════════════╣");
      Console.WriteLine($"║ Brakes:  {(_brakesApplied ? "[APPLIED]" : "[ OFF ]"),9}         ║");
      Console.WriteLine($"║ ABS:     {(_absActive ? "[ACTIVE]" : "[  OFF ]"),9}         ║");
      Console.WriteLine("╠════════════════════════════════════╣");
      Console.WriteLine("║              WARNING LIGHTS            ║");
      Console.WriteLine($"║ Check Engine: {(_checkEngineLight ? "[ON ]" : "[OFF]")}           ║");
      Console.WriteLine($"║ Overheating:  {(_overheatingWarning ? "[ON ]" : "[OFF]")}           ║");
      Console.WriteLine($"║ Low Fuel:     {(_lowFuelWarning ? "[ON ]" : "[OFF]")}           ║");
      Console.WriteLine("╚════════════════════════════════════╝\n");
    }

    static async Task HandleUserInput()
    {
      while (_running)
      {
        var key = Console.ReadKey(true);

        switch (key.KeyChar)
        {
          case 'f':
            _fuelLevel = Math.Max(0, _fuelLevel - 10);
            _lowFuelWarning = _fuelLevel < 15;
            Console.WriteLine($"Fuel decreased to: {_fuelLevel}%");
            break;

          case 'F':
            _fuelLevel = Math.Min(100, _fuelLevel + 10);
            _lowFuelWarning = _fuelLevel < 15;
            Console.WriteLine($"Fuel increased to: {_fuelLevel}%");
            break;

          case 'w':
            _checkEngineLight = !_checkEngineLight;
            Console.WriteLine($"Check Engine Light: {(_checkEngineLight ? "ON" : "OFF")}");
            break;

          case 'd':
            ShowDashboard();
            break;

          case 'q':
            _running = false;
            Console.WriteLine("Shutting down Dashboard ECU...");
            return;
        }
      }
    }
  }
}

```

### ABSECU
```csharp
using System;
using System.Threading.Tasks;
using System.Net.Sockets;
using System.Runtime.InteropServices;
using SocketCANSharp;

namespace ABSECU
{
    class Program
    {
        private static bool _running = true;
        private static SafeFileDescriptorHandle _socketHandle;

        // ABS parameters
        private static int _wheelSpeed = 0;             // km/h - derived from engine RPM and gear
        private static bool _brakePedalPressed = false; // Brake pedal status
        private static bool _absActive = false;         // ABS intervention active
        private static int _brakeForce = 0;             // 0-100% brake force applied

        static async Task Main(string[] args)
        {
            Console.WriteLine("=== ABS ECU SIMULATOR ===");
            Console.WriteLine("Sending: Wheel Speed, Brake Status, ABS Status");
            Console.WriteLine("Listening: Engine RPM, Transmission Gear");
            Console.WriteLine();

            try
            {
                await InitializeCAN();

                Console.WriteLine("Commands:");
                Console.WriteLine("  'b'       : Toggle brake pedal");
                Console.WriteLine("  '+' / '-' : Increase/Decrease brake force");
                Console.WriteLine("  'a'       : Toggle ABS system");
                Console.WriteLine("  'q'       : Quit");
                Console.WriteLine();

                // Start background tasks
                var sendTask = Task.Run(SendABSData);
                var listenTask = Task.Run(ListenToCANBus);
                var inputTask = Task.Run(HandleUserInput);

                await Task.WhenAny(inputTask);
                _running = false;

                await Task.WhenAll(sendTask, listenTask);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
            }
            finally
            {
                _socketHandle?.Dispose();
            }
        }

        static async Task InitializeCAN()
        {
            _socketHandle = LibcNativeMethods.Socket(SocketCanConstants.PF_CAN, SocketType.Raw, SocketCanProtocolType.CAN_RAW);
            if (_socketHandle.IsInvalid)
                throw new Exception("Failed to create CAN socket");

            var ifr = new Ifreq("vcan0");
            int ioctlResult = LibcNativeMethods.Ioctl(_socketHandle, SocketCanConstants.SIOCGIFINDEX, ifr);
            if (ioctlResult == -1)
                throw new Exception("vcan0 interface not found");

            var addr = new SockAddrCan(ifr.IfIndex);
            int bindResult = LibcNativeMethods.Bind(_socketHandle, addr, Marshal.SizeOf(typeof(SockAddrCan)));
            if (bindResult == -1)
                throw new Exception("Failed to bind to vcan0");

            Console.WriteLine("✓ Connected to CAN bus (vcan0) as ABS ECU");
        }

        static async Task SendABSData()
        {
            while (_running)
            {
                try
                {
                    // Send wheel speed every 50ms (20Hz) - High frequency for safety
                    SendWheelSpeed();
                    await Task.Delay(5000);

                    // Send brake pedal status every 100ms (10Hz)
                    if (DateTime.Now.Millisecond % 100 < 50)
                        SendBrakePedalStatus();

                    // Send ABS status every 200ms (5Hz)
                    if (DateTime.Now.Millisecond % 200 < 50)
                        SendABSStatus();
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Send error: {ex.Message}");
                    await Task.Delay(10000);
                }
            }
        }

        static void SendWheelSpeed()
        {
            var frame = new CanFrame
            {
                CanId = 0x220, // Wheel Speed
                Length = 2,
                Data = new byte[8]
            };

            // Pack wheel speed as 16-bit little endian
            frame.Data[0] = (byte)(_wheelSpeed & 0xFF);
            frame.Data[1] = (byte)((_wheelSpeed >> 8) & 0xFF);

            SendCANFrame(frame, $"WHEEL SPEED: {_wheelSpeed} km/h");
        }

        static void SendBrakePedalStatus()
        {
            var frame = new CanFrame
            {
                CanId = 0x221, // Brake Pedal Status
                Length = 2,
                Data = new byte[8]
            };

            frame.Data[0] = (byte)(_brakePedalPressed ? 1 : 0);
            frame.Data[1] = (byte)_brakeForce;

            SendCANFrame(frame, $"BRAKE: {(_brakePedalPressed ? "PRESSED" : "RELEASED")} ({_brakeForce}%)");
        }

        static void SendABSStatus()
        {
            var frame = new CanFrame
            {
                CanId = 0x222, // ABS Status
                Length = 1,
                Data = new byte[8]
            };

            frame.Data[0] = (byte)(_absActive ? 1 : 0);
            SendCANFrame(frame, $"ABS: {(_absActive ? "ACTIVE" : "INACTIVE")}");
        }

        static void SendCANFrame(CanFrame frame, string description)
        {
            try
            {
                int frameSize = Marshal.SizeOf(typeof(CanFrame));
                int bytesWritten = LibcNativeMethods.Write(_socketHandle, ref frame, frameSize);

                if (bytesWritten > 0)
                {
                    Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] TX → 0x{frame.CanId:X3}: {description}");
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Failed to send {description}: {ex.Message}");
            }
        }

        static async Task ListenToCANBus()
        {
            int frameSize = Marshal.SizeOf(typeof(CanFrame));
            int currentEngineRpm = 800;
            int currentGear = 1;

            while (_running)
            {
                try
                {
                    var frame = new CanFrame();
                    int bytesRead = LibcNativeMethods.Read(_socketHandle, ref frame, frameSize);

                    if (bytesRead > 0)
                    {
                        ProcessIncomingMessage(frame, ref currentEngineRpm, ref currentGear);
                    }
                }
                catch (Exception ex)
                {
                    if (_running)
                        Console.WriteLine($"Listen error: {ex.Message}");
                }

                await Task.Delay(1000);
            }
        }

        static void ProcessIncomingMessage(CanFrame frame, ref int engineRpm, ref int gear)
        {
            string timestamp = DateTime.Now.ToString("HH:mm:ss.fff");

            switch (frame.CanId)
            {
                case 0x110: // Engine RPM
                    engineRpm = frame.Data[0] | (frame.Data[1] << 8);
                    Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: ENGINE RPM {engineRpm}");

                    CalculateWheelSpeed(engineRpm, gear);
                    break;

                case 0x330: // Transmission Gear
                    gear = frame.Data[0];
                    Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: GEAR {gear}");

                    CalculateWheelSpeed(engineRpm, gear);
                    break;

                case 0x113: // Engine Status
                    bool engineRunning = frame.Data[0] == 1;
                    Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: ENGINE {(engineRunning ? "ON" : "OFF")}");

                    if (!engineRunning)
                        _wheelSpeed = 0;
                    break;

                default:
                    Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: Other ECU message");
                    break;
            }
        }

        static void CalculateWheelSpeed(int engineRpm, int gear)
        {
            double[] gearRatios = { 0, 3.5, 2.1, 1.4, 1.0, 0.8, 0.6 };

            if (gear >= 1 && gear <= 6 && engineRpm > 0)
            {
                _wheelSpeed = Math.Max(0, (int)((engineRpm / gearRatios[gear]) / 25));

                if (_brakePedalPressed)
                {
                    int speedReduction = (_brakeForce * _wheelSpeed) / 100;
                    _wheelSpeed = Math.Max(0, _wheelSpeed - speedReduction);
                    _absActive = _brakeForce > 50 && _wheelSpeed > 30;
                }
                else
                {
                    _absActive = false;
                }
            }
            else
            {
                _wheelSpeed = 0;
                _absActive = false;
            }
        }

        static async Task HandleUserInput()
        {
            while (_running)
            {
                var key = Console.ReadKey(true);

                switch (key.KeyChar)
                {
                    case 'b':
                        _brakePedalPressed = !_brakePedalPressed;
                        if (_brakePedalPressed)
                        {
                            _brakeForce = 30;
                        }
                        else
                        {
                            _brakeForce = 0;
                            _absActive = false;
                        }
                        Console.WriteLine($"Brake pedal: {(_brakePedalPressed ? "PRESSED" : "RELEASED")}");
                        break;

                    case '+':
                        if (_brakePedalPressed)
                        {
                            _brakeForce = Math.Min(100, _brakeForce + 20);
                            Console.WriteLine($"Brake force increased to: {_brakeForce}%");
                        }
                        break;

                    case '-':
                        if (_brakePedalPressed)
                        {
                            _brakeForce = Math.Max(0, _brakeForce - 20);
                            Console.WriteLine($"Brake force decreased to: {_brakeForce}%");
                        }
                        break;

                    case 'a':
                        _absActive = !_absActive;
                        Console.WriteLine($"ABS manually set to: {(_absActive ? "ACTIVE" : "INACTIVE")}");
                        break;

                    case 'q':
                        _running = false;
                        Console.WriteLine("Shutting down ABS ECU...");
                        return;
                }
            }
        }
    }
}

```

### TransmissionECU
```csharp
using System;
using System.Threading.Tasks;
using System.Net.Sockets;
using System.Runtime.InteropServices;
using SocketCANSharp;

namespace TransmissionECU
{
    class Program
    {
        private static bool _running = true;
        private static SafeFileDescriptorHandle _socketHandle;

        // Transmission parameters
        private static int _currentGear = 1;           // Current gear (1-6)
        private static int _transmissionTemp = 75;     // Celsius
        private static bool _transmissionReady = true; // Ready status
        private static int _lastEngineRpm = 800;       // For auto-shifting logic

        static async Task Main(string[] args)
        {
            Console.WriteLine("=== TRANSMISSION ECU SIMULATOR ===");
            Console.WriteLine("Sending: Gear, Temperature, Status");
            Console.WriteLine("Listening: Engine RPM (for auto-shift)");
            Console.WriteLine();

            try
            {
                await InitializeCAN();

                Console.WriteLine("Commands:");
                Console.WriteLine("  'u' / 'd' : Manual gear up/down");
                Console.WriteLine("  'h' / 'c' : Heat up/Cool down transmission");
                Console.WriteLine("  't'       : Toggle transmission ready");
                Console.WriteLine("  'a'       : Toggle auto-shift mode");
                Console.WriteLine("  'q'       : Quit");
                Console.WriteLine();

                // Start background tasks
                var sendTask = Task.Run(SendTransmissionData);
                var listenTask = Task.Run(ListenToCANBus);
                var inputTask = Task.Run(HandleUserInput);

                await Task.WhenAny(inputTask);
                _running = false;

                await Task.WhenAll(sendTask, listenTask);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
            }
            finally
            {
                _socketHandle?.Dispose();
            }
        }

        static async Task InitializeCAN()
        {
            _socketHandle = LibcNativeMethods.Socket(SocketCanConstants.PF_CAN, SocketType.Raw, SocketCanProtocolType.CAN_RAW);
            if (_socketHandle.IsInvalid)
                throw new Exception("Failed to create CAN socket");

            var ifr = new Ifreq("vcan0");
            int ioctlResult = LibcNativeMethods.Ioctl(_socketHandle, SocketCanConstants.SIOCGIFINDEX, ifr);
            if (ioctlResult == -1)
                throw new Exception("vcan0 interface not found");

            var addr = new SockAddrCan(ifr.IfIndex);
            int bindResult = LibcNativeMethods.Bind(_socketHandle, addr, Marshal.SizeOf(typeof(SockAddrCan)));
            if (bindResult == -1)
                throw new Exception("Failed to bind to vcan0");

            Console.WriteLine("✓ Connected to CAN bus (vcan0) as TRANSMISSION ECU");
        }

        static async Task SendTransmissionData()
        {
            while (_running)
            {
                try
                {
                    // Send current gear every 20s
                    SendCurrentGear();
                    await Task.Delay(20000);

                    // Send transmission temperature every 2000ms (0.5Hz)
                    if (DateTime.Now.Millisecond < 200)
                        SendTransmissionTemperature();

                    // Send transmission status every 1000ms (1Hz)
                    if (DateTime.Now.Millisecond % 1000 < 200)
                        SendTransmissionStatus();
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Send error: {ex.Message}");
                    await Task.Delay(1000);
                }
            }
        }

        static void SendCurrentGear()
        {
            var frame = new CanFrame
            {
                CanId = 0x330, // Current Gear
                Length = 1,
                Data = new byte[8]
            };

            frame.Data[0] = (byte)_currentGear;
            SendCANFrame(frame, $"GEAR: {_currentGear}");
        }

        static void SendTransmissionTemperature()
        {
            var frame = new CanFrame
            {
                CanId = 0x331, // Transmission Temperature
                Length = 1,
                Data = new byte[8]
            };

            frame.Data[0] = (byte)_transmissionTemp;
            SendCANFrame(frame, $"TRANS TEMP: {_transmissionTemp}°C");
        }

        static void SendTransmissionStatus()
        {
            var frame = new CanFrame
            {
                CanId = 0x332, // Transmission Status
                Length = 1,
                Data = new byte[8]
            };

            frame.Data[0] = (byte)(_transmissionReady ? 1 : 0);
            SendCANFrame(frame, $"TRANS STATUS: {(_transmissionReady ? "READY" : "NOT READY")}");
        }

        static void SendCANFrame(CanFrame frame, string description)
        {
            try
            {
                int frameSize = Marshal.SizeOf(typeof(CanFrame));
                int bytesWritten = LibcNativeMethods.Write(_socketHandle, ref frame, frameSize);

                if (bytesWritten > 0)
                {
                    Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] TX → 0x{frame.CanId:X3}: {description}");
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Failed to send {description}: {ex.Message}");
            }
        }

        static async Task ListenToCANBus()
        {
            int frameSize = Marshal.SizeOf(typeof(CanFrame));

            while (_running)
            {
                try
                {
                    var frame = new CanFrame();
                    int bytesRead = LibcNativeMethods.Read(_socketHandle, ref frame, frameSize);

                    if (bytesRead > 0)
                        ProcessIncomingMessage(frame);
                }
                catch (Exception ex)
                {
                    if (_running)
                        Console.WriteLine($"Listen error: {ex.Message}");
                }

                await Task.Delay(1000);
            }
        }

        static void ProcessIncomingMessage(CanFrame frame)
        {
            string timestamp = DateTime.Now.ToString("HH:mm:ss.fff");

            switch (frame.CanId)
            {
                case 0x110: // Engine RPM
                    int engineRpm = frame.Data[0] | (frame.Data[1] << 8);
                    Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: ENGINE RPM {engineRpm}");
                    CheckAutoShift(engineRpm);
                    _lastEngineRpm = engineRpm;
                    break;

                case 0x113: // Engine Status
                    bool engineRunning = frame.Data[0] == 1;
                    Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: ENGINE {(engineRunning ? "ON" : "OFF")}");
                    if (!engineRunning && _currentGear != 1)
                    {
                        _currentGear = 1;
                        Console.WriteLine("            → Reset to gear 1 (engine stopped)");
                    }
                    break;

                case 0x220: // Brake signal
                    bool brakesApplied = frame.Data[0] == 1;
                    Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: BRAKE {(brakesApplied ? "APPLIED" : "RELEASED")}");
                    break;

                default:
                    Console.WriteLine($"[{timestamp}] RX ← 0x{frame.CanId:X3}: Other ECU message");
                    break;
            }
        }

        static void CheckAutoShift(int engineRpm)
        {
            int newGear = _currentGear;

            if (engineRpm > 3500 && _currentGear < 6)
                newGear = _currentGear + 1;
            else if (engineRpm < 1200 && _currentGear > 1)
                newGear = _currentGear - 1;

            if (newGear != _currentGear)
            {
                _currentGear = newGear;
                Console.WriteLine($"            → AUTO-SHIFT to gear {_currentGear} (RPM: {engineRpm})");
            }
        }

        static async Task HandleUserInput()
        {
            bool autoShiftMode = true;
            Console.WriteLine($"Auto-shift mode: {(autoShiftMode ? "ON" : "OFF")}");

            while (_running)
            {
                var key = Console.ReadKey(true);

                switch (key.KeyChar)
                {
                    case 'u':
                        if (_currentGear < 6)
                        {
                            _currentGear++;
                            Console.WriteLine($"Manual shift UP to gear: {_currentGear}");
                        }
                        break;

                    case 'd':
                        if (_currentGear > 1)
                        {
                            _currentGear--;
                            Console.WriteLine($"Manual shift DOWN to gear: {_currentGear}");
                        }
                        break;

                    case 'h':
                        _transmissionTemp = Math.Min(120, _transmissionTemp + 10);
                        Console.WriteLine($"Transmission temp: {_transmissionTemp}°C");
                        break;

                    case 'c':
                        _transmissionTemp = Math.Max(40, _transmissionTemp - 10);
                        Console.WriteLine($"Transmission temp: {_transmissionTemp}°C");
                        break;

                    case 't':
                        _transmissionReady = !_transmissionReady;
                        Console.WriteLine($"Transmission: {(_transmissionReady ? "READY" : "NOT READY")}");
                        break;

                    case 'a':
                        autoShiftMode = !autoShiftMode;
                        Console.WriteLine($"Auto-shift mode: {(autoShiftMode ? "ON" : "OFF")}");
                        break;

                    case 'q':
                        _running = false;
                        Console.WriteLine("Shutting down Transmission ECU...");
                        return;
                }
            }
        }
    }
}

```