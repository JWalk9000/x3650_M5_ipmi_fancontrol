# IPMI Fan Control Daemon - AI Coding Instructions

## Project Overview
This is a Perl daemon that controls IBM System x3650 M5 server fan speeds via IPMI commands based on CPU temperatures. It's a fork specifically adapted for systems with 6 fans (1A-6A) and 2 fan banks.

## Key Architecture

### Core Components
- **`ipmi_fancontrol-ng`**: Main Perl daemon implementing temperature-based fan curve control
- **`ipmi_fancontrol-ng.service`**: SystemD service definition for daemon management
- **Fan curve algorithm**: Uses linear interpolation (`Y=mx+b`) between temperature/speed points instead of step-based control

### Critical Configuration Points
```perl
# System-specific settings that must match hardware
my $number_of_cpus      = 2;  # IBM x3650 M5 dual CPU config
my $number_of_fans      = 6;  # All fans labeled 1A-6A (no B fans)
my $number_of_fanbanks  = 2;  # Hardware limitation despite single-letter naming
```

### IPMI Command Pattern
```perl
# IBM-specific raw IPMI command for fan control:
# ipmitool raw 0x3a 0x07 <bank_id> <speed_percent> 0x01
`$ipmi_preamble raw 0x3a 0x07 $i $fan_speed 0x01> /dev/null 2>&1`;
```

## Hardware-Specific Constraints

### IBM IMM2 Behavior
- IMM2 maintains partial fan control override
- Setting 30% may result in actual ~24% speed
- IMM2 can reclaim control if script fails to update
- Fan speed control is "coarse" - limited precision

### Temperature Sources
- Uses `lm-sensors` with "Package" temperature readings
- Automatically selects highest CPU temperature when multiple CPUs present
- No GPU temperature monitoring (commented out but structure remains)

## Development Workflows

### Testing & Debugging
```bash
# Manual execution for testing
sudo ./ipmi_fancontrol-ng

# Check IPMI modules
lsmod | grep -i ipmi

# Monitor service logs
journalctl -u ipmi_fancontrol-ng -f
```

### Service Management
```bash
# Installation sequence
cp ipmi_fancontrol-ng /usr/bin/
cp ipmi_fancontrol-ng.service /usr/lib/systemd/system/
chmod +x /usr/bin/ipmi_fancontrol-ng
systemctl enable ipmi_fancontrol-ng
systemctl start ipmi_fancontrol-ng
```

## Monitoring Integration

### InfluxDB/Telegraf Output
- Metrics written to `/tmp/fan_speed_telegraf` in InfluxDB line protocol
- Provides both percentage and hex values for fan speeds
- Hostname-tagged for multi-server environments

#### Useful Diagnostics (manual)
```bash
ipmitool sdr info
ipmitool sdr type Fan
ipmitool sel elist | tail -n 20
```

## Code Patterns

### Fan Curve Calculation
The script pre-calculates linear equations between temperature points in `%cpu_temp_to_fan_speed` hash:
```perl
# Temperature -> Speed mappings (customize for thermal requirements)
$cpu_temp_to_fan_speed{80} = 250;  # Emergency high speed
$cpu_temp_to_fan_speed{40} = 25;   # Idle target (40°C at 25%/~1888RPM)
```

### Control Resilience (IMM2 quirks)
- Periodic reassert: `$refresh_period_secs` forces a fan command at least every N seconds to prevent IMM2 reclaim
- Clamp bounds: `$min_duty`/`$max_duty` restrict requested duty (0–255) to avoid values that provoke ramp-ups
- Override detection: compares requested duty to actual tach readings via `ipmitool sdr type Fan`; if RPM spikes (e.g., >8000 RPM while duty <80), the daemon reasserts the command and logs a warning
- Diagnostics: set `$enable_diagnostics=1` to emit every `$diagnostics_every_loops` cycles: fan RPMs, `sdr info`, and tail of `sel elist`

### Error Handling
- IPMI commands redirected to `/dev/null` (fire-and-forget approach)
- Service configured with automatic restart on failure
- No explicit error checking on IPMI communication

## Important Notes
- **Safety**: Script puts server in "Full Fan Speed Mode" then redefines "full speed"
- **Recovery**: Manual IPMI commands needed to return to "Optimal" mode when script stops
- **Environment**: Originally adapted for Unraid via User Scripts but works with SystemD
- **Dependencies**: Requires `lm-sensors` and `ipmitool` packages