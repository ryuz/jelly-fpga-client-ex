# JellyFpgaClient

## Installation

### FPGA Server Side (ex. AMD KV260 Ubuntu 22.04)

Install Rust if it is not already installed.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Install jelly-fpga-server.

```bash
cargo install --git https://github.com/ryuz/jelly-fpga-server.git --tag v0.0.4
```

Start the server

```bash
sudo $HOME/.cargo/bin/jelly-fpga-server --external
```

### Client Side

The package can be installed by adding `jelly_fpga_client` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:jelly_fpga_client, github: "ryuz/jelly-fpga-client-elixir"}
  ]
end
```

## Usage

The following is an example of [KV260 LED blinking](https://github.com/ryuz/jelly/tree/master/projects/kv260/kv260_blinking_led_ps).

Connect to FPGA.

```elixir
{:ok, channel} = GRPC.Stub.connect("XX.XX.XX.XX:8051")
channel |> JellyFpgaControl.reset()
```

Compile the DTS.

```elixir
dts = """
/dts-v1/; /plugin/;
/ {
  fragment@0 {
    target = <&fpga_full>;
    overlay0: __overlay__ {
      #address-cells = <2>;
      #size-cells = <2>;
      firmware-name = "kv260_blinking_led_ps.bit.bin";
    };
  };

  fragment@1 {
    target = <&amba>;
    overlay1: __overlay__ {
      clocking0: clocking0 {
        #clock-cells = <0>;
        assigned-clock-rates = <100000000>;
        assigned-clocks = <&zynqmp_clk 71>;
        clock-output-names = "fabric_clk";
        clocks = <&zynqmp_clk 71>;
        compatible = "xlnx,fclk";
      };
    };
  };
};
"""

# Convert to DTB and upload as firmware
{:ok, dtb} = channel |> JellyFpgaControl.dts_to_dtb(dts)
channel |> JellyFpgaControl.upload_firmware("kv260_blinking_led_ps.dtbo", dtb)
```

Next, upload the bitstream and convert it to bin.

```elixir
# Upload the bitstream file
channel |> JellyFpgaControl.upload_firmware_file("kv260_blinking_led_ps.bit", "./kv260_blinking_led_ps.bit")
# Convert the uploaded bitstream file to a bin file
channel |> JellyFpgaControl.bitstream_to_bin("kv260_blinking_led_ps.bit", "kv260_blinking_led_ps.bit.bin", "zynqmp")
```

Apply the DeviceTree Overlay and load it into the PL.

```elixir
# Unload the existing PL
channel |> JellyFpgaControl.unload()
# Apply the DeviceTree Overlay
channel |> JellyFpgaControl.load_dtbo("kv260_blinking_led_ps.dtbo")
```

Use mmap on /dev/mem to blink the LED.

```elixir
# Use mmap on /dev/mem to blink LED0
{:ok, accessor} = channel |> JellyFpgaControl.open_mmap("/dev/mem", 0xa0000000, 0x1000)
Enum.each(1..3, fn _ ->
  # LED0 ON
  channel |> JellyFpgaControl.write_mem_u64(accessor, 0, 1) # Write 1 to offset 0
  Process.sleep(500)
  # LED0 OFF
  channel |> JellyFpgaControl.write_mem_u64(accessor, 0, 0) # Write 0 to offset 0
  Process.sleep(500)
  end)
```

Clean up.

```elixir
# Clean up
channel |> JellyFpgaControl.unload()
channel |> JellyFpgaControl.remove_firmware("kv260_blinking_led_ps.dtbo")
channel |> JellyFpgaControl.remove_firmware("kv260_blinking_led_ps.bit")
channel |> JellyFpgaControl.remove_firmware("kv260_blinking_led_ps.bit.bin")
```

