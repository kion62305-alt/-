  from flask import Flask, jsonify, render_template, abort
from gpiozero import OutputDevice, Button
from pathlib import Path
import threading
import time
import glob
import serial

def find_serial_port():
    ports = glob.glob("/dev/ttyUSB*") + glob.glob("/dev/ttyACM*")
    return ports[0] if ports else None

def send_arduino_command(locker_id, mode):
    port = find_serial_port()

    if not port:
        print("arduino port not found")
        return

    if mode == "rent":
        command = f"{locker_id}:RENT\n"
    elif mode == "return":
        command = f"{locker_id}:RETURN\n"
    else:
        return

    try:
        ser = serial.Serial(port, 9600, timeout=1)
        time.sleep(2)
        ser.write(command.encode("utf-8"))
        ser.flush()
        ser.close()
        print("sent:", command.strip())

    except Exception as e:
        print("arduino send error:", e)

app = Flask(__name__, template_folder="templates")

solenoids = {
    "BOX_1": OutputDevice(18, active_high=True, initial_value=False),
    "BOX_2": OutputDevice(17, active_high=True, initial_value=False),
}

reed_switches = {
    "BOX_1": Button(25, pull_up=True),
    "BOX_2": Button(24, pull_up=True),
}

ALLOWED_PAGES = {
    "index.html",
    "login_my.html",
    "map_my.html",
    "qr_my.html",
    "reservation_my.html",
    "signup_my.html",
}

locker_open_state = {
    "BOX_1": False,
    "BOX_2": False,
}


@app.route("/")
def home():
    return render_template("index.html")

@app.route("/<page_name>")
def serve_page(page_name):
    if page_name not in ALLOWED_PAGES:
        abort(404)
    return render_template(page_name)

@app.route("/open/<mode>/<locker_id>", methods=["GET"])
def open_locker(mode, locker_id):
    mode = mode.strip().lower()
    locker_id = locker_id.strip().upper()

    if mode not in ["rent", "return"]:
        return jsonify({"success": False, "message": "invalid mode"}), 400

    device = solenoids.get(locker_id)

    if device is None:
        return jsonify({"success": False, "message": f"{locker_id} not found"}), 404

    try:
        device.on()
        send_arduino_command(locker_id, mode)

        locker_open_state[locker_id] = False
        threading.Timer(
            5,
            lambda lid=locker_id: locker_open_state.__setitem__(lid, True)
        ).start()

        return jsonify({
            "success": True,
            "message": f"{locker_id} opened",
            "mode": mode
        }), 200

    except Exception as e:
        try:
            device.off()
        except Exception:
            pass

        locker_open_state[locker_id] = False
        return jsonify({"success": False, "message": str(e)}), 500


def monitor_reed_switch():
    while True:
        for locker_id, switch in reed_switches.items():
            if locker_open_state[locker_id]:
                if switch.is_pressed:
                    print(f"{locker_id} closed detected -> solenoid OFF")
                    solenoids[locker_id].off()
                    locker_open_state[locker_id] = False

        time.sleep(0.1)


@app.route("/status/<locker_id>", methods=["GET"])
def status_locker(locker_id):
    locker_id = locker_id.strip().upper()

    if locker_id not in solenoids:
        return jsonify({"success": False, "message": "locker not found"}), 404

    return jsonify({
        "success": True,
        "locker_id": locker_id,
        "reed_detected": reed_switches[locker_id].is_pressed,
        "solenoid_open": locker_open_state[locker_id]
    }), 200


if __name__ == "__main__":
    monitor_thread = threading.Thread(target=monitor_reed_switch, daemon=True)
    monitor_thread.start()

    cert_file = Path("cert.pem")
    key_file = Path("key.pem")

    if cert_file.exists() and key_file.exists():
        app.run(host="0.0.0.0", port=5000, ssl_context=("cert.pem", "key.pem"))
    else:
        app.run(host="0.0.0.0", port=5000, ssl_context="adhoc")
