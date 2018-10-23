#!/usr/bin/env python3

import argparse
import json
import logging
import os
import os.path
import signal
import socket
import subprocess
import time

import gi
gi.require_version('Gtk', '3.0')  # noqa
from gi.repository import Gtk

CONNECT_RETRY_DELAY = 2.0


LOGGER = logging.getLogger(__name__)
WORK_DIR = os.getcwd()

post_menu_action = None


class RPC:
    def __init__(self, path):
        self.path = path
        self.socket = socket.socket(socket.AF_UNIX,
                                    socket.SOCK_STREAM)
        connected = False
        while not connected:
            try:
                logging.debug("Attempting to connect to RPC.")
                self.socket.connect(self.path)
                connected = True
            except socket.error:
                logging.debug("Connection attempt failed.")
                time.sleep(CONNECT_RETRY_DELAY)
        self.file = self.socket.makefile("rw", 65536)

    def send_line(self, s):
        self.file.write(s + "\n")
        self.file.flush()

    def recv_line(self):
        return self.file.readline(1024)

    def send_cmd(self, *args):
        self.send_line(json.dumps({"command": args}))

    def recv_data(self):
        s = self.recv_line()
        if not s:
            return None
        return json.loads(s)

    def get_result(self):
        dat = {}
        while not ("error" in dat):
            dat = self.recv_data()
        return dat

    def command(self, *args):
        self.send_cmd(*args)
        dat = self.get_result()
        success = (dat["error"] == "success")
        return (success, dat["data"] if success else None)

    def set_prop(self, prop, value):
        self.send_cmd("set_property", prop, value)
        return self.get_result()["error"] == "success"

    def get_prop(self, prop):
        self.send_cmd("get_property", prop)
        dat = self.get_result()
        return dat["data"] if dat["error"] == "success" else None


class OPT:
    NORMAL = 0
    CHECK = 1
    SEP = 2
    SLIDER = 3

    def __init__(self, name=None, typ=NORMAL,
                 init=None, activate=None):
        self.name = name
        self.typ = typ
        self.init_ = init
        self.activate_ = activate

    def init(self):
        if self.init_:
            return self.init_()

    def activate(self):
        if self.activate_:
            return self.activate_()
        else:
            LOGGER.debug("Option {} activated.".format(self.name))


SEP = OPT(typ=OPT.SEP)


class TOGGLE(OPT):
    def __init__(self, name, prop):
        OPT.__init__(self, name, OPT.CHECK)
        self.prop = prop

    def init(self):
        self.state = rpc.get_prop(self.prop)
        # In case we got None, because this property "vanished"
        # for some reason, default to False.
        if self.state is None:
            LOGGER.warn("Can't get value for toggle property {}."
                        " No need to panic though.".format(self.prop))
            self.state = False
        LOGGER.debug("initial state for {} : {}."
                     .format(self.prop, self.state))

    def activate(self):
        rpc.set_prop(self.prop, not self.state)
        LOGGER.debug("Property {} change: {} -> {}."
                     .format(self.prop, self.state, not self.state))


class FILTER_TOGGLE(OPT):
    def __init__(self, name, filter_type, filter_name, filter_opts):
        OPT.__init__(self, name, OPT.CHECK)
        self.ft = "af" if filter_type[0] == "a" else "vf"
        self.filter_name = filter_name
        self.filter_opts = filter_opts

    def init(self):
        self.state = (self.filter_name in map(lambda x: x["name"],
                                              rpc.get_prop(self.ft)))

    def activate(self):
        # Note: af/vf toggle command might be removed/changed later.
        rpc.command(self.ft, "toggle", self.filter_name+"="+self.filter_opts)


class COMMAND(OPT):
    def __init__(self, name, *args):
        OPT.__init__(self, name, OPT.NORMAL)
        self.args = args

    def activate(self):
        rpc.command(*self.args)


class OPT_SET_PROP(OPT):
    def __init__(self, name, prop, value):
        OPT.__init__(self, name, OPT.NORMAL)
        self.prop = prop
        self.value = value

    def activate(self):
        rpc.set_prop(self.prop, self.value)


def get_track_info():
    info = {"video": [], "audio": [], "sub": []}
    for i in range(rpc.get_prop("track-list/count")):
        N = str(i)
        type_ = rpc.get_prop("track-list/" + N + "/type")
        track = {
            "id": rpc.get_prop("track-list/" + N + "/id"),
            "src-id": rpc.get_prop("track-list/" + N + "/src-id"),
            "title": rpc.get_prop("track-list/" + N + "/title"),
            "lang": rpc.get_prop("track-list/" + N + "/lang"),
            "default": rpc.get_prop("track-list/" + N + "/default"),
        }
        if rpc.get_prop("track-list/" + N + "/external"):
            track["filename"] = rpc.get_prop("track-list/" + N +
                                             "/external-filename")
        info[type_].append(track)
    return info


def to_abs_path(path):
    return os.path.normpath(os.path.join(WORK_DIR, path))


def load_file_run_dialog(title):
    dialog = Gtk.FileChooserDialog(title, None,
                                   Gtk.FileChooserAction.OPEN,
                                   (Gtk.STOCK_CANCEL, Gtk.ResponseType.CANCEL,
                                    Gtk.STOCK_OPEN, Gtk.ResponseType.OK))
    dialog.set_current_folder(os.path.dirname(
                              to_abs_path(rpc.get_prop("path"))))
    filename = ""
    if (dialog.run() == Gtk.ResponseType.OK):
        filename = dialog.get_filename()
    dialog.destroy()
    Gtk.main_quit()
    return filename


def load_sub_file():
    filename = load_file_run_dialog("Choose a subtitle file")
    if filename:
        rpc.command("sub_add", filename)
        LOGGER.debug("Subtitle file to load: {}.".format(filename))
    else:
        LOGGER.debug("Subtitle file load dlg canceled.")


def load_file():
    filename = load_file_run_dialog("Choose a subtitle file")
    if filename:
        rpc.command("loadfile", filename)


def post_menu_action_factory(act):
    def func():
        global post_menu_action
        post_menu_action = act
    return func


def dl_subs():
    path = to_abs_path(rpc.get_prop("path"))
    try:
        logging.debug("Calling subdownloader.")
        status = subprocess.call(["subdownloader", "--rename-video",
                                  "-V", path])
        logging.debug("Subdownloader exited with status: {}".format(status))
        rpc.command("rescan-external-files")
    except OSError:
        dialog = Gtk.MessageDialog(None, 0, Gtk.MessageType.INFO,
                                   Gtk.ButtonsType.OK,
                                   "Subdownloader required")
        dialog.format_secondary_text("Currently, this option requires" +
                                     "\nsubdownloader to be installed.")
        dialog.run()
        dialog.destroy()
        Gtk.main_quit()


def about():
    dialog = Gtk.AboutDialog(program_name="MPVMenu",
                             comments="Popup menu for MPV",
                             logo_icon_name="")
    dialog.run()
    dialog.destroy()
    Gtk.main_quit()


class Layout:
    def __init__(self, *args):
        if len(args) > 1 and isinstance(args[0], str):
            self.name = args[0]
            args = args[1:]
        self.items = args


class TracklistLayout(Layout):
    TYPE_VIDEO = 0
    TYPE_AUDIO = 1
    TYPE_SUB = 2

    def __init__(self, name, typ):
        self.typ = typ
        self.name = name

    @property
    def items(self):
        typ = ("video", "audio", "sub")[self.typ]
        tracklist = get_track_info()[typ]
        return list(map(self.track_info_to_opt, tracklist))

    def track_info_to_opt(self, track):
        prop = ("vid", "aid", "sid")[self.typ]
        id_ = track["id"]
        title = " "+track["title"] if track["title"] else " Untitled"
        lang = " ("+track["lang"]+")" if track["lang"] else ""
        default = " (default)" if track["default"] else ""
        name = "{}{}{}{}".format(id_, title,
                                 lang, default)
        return OPT_SET_PROP(name, prop, track["id"])


layout = Layout(
    Layout(
        "File",
        OPT("Open file", activate=load_file),
        SEP,
        COMMAND("Quit mpv", "quit"),
        COMMAND("Quit mpv (watch later)", "quit_watch_later"),
        OPT("Quit mpvmenu", activate=exit)
    ),
    Layout(
        "Playback",
        TOGGLE("Pause", "pause"),
        Layout(
            "Rewind",
            COMMAND("3 seconds", "seek", "-3"),
            COMMAND("10 seconds", "seek", "-10"),
            COMMAND("1 minute", "seek", "-60")
        ),
        Layout(
            "Fast forward",
            COMMAND("3 seconds", "seek", "3"),
            COMMAND("10 seconds", "seek", "10"),
            COMMAND("1 minute", "seek", "60")
        ),
    ),
    Layout(
        "Playlist",
        COMMAND("Previous", "playlist_prev"),
        COMMAND("Next", "playlist_next")
    ),
    Layout(
        "Audio",
        TracklistLayout("Select audio track",
                        TracklistLayout.TYPE_AUDIO),
        TOGGLE("Mute", "mute"),
        Layout(
            "Audio Filters",
            FILTER_TOGGLE("Dynamic Range Compression",
                          "a", "drc", "2:1")
        )
    ),
    Layout(
        "Video",
        TracklistLayout("Select video track",
                        TracklistLayout.TYPE_VIDEO),
        TOGGLE("Fullscreen", "fullscreen")
    ),
    Layout(
        "Subtitles",
        TracklistLayout("Select subtitle track",
                        TracklistLayout.TYPE_SUB),
        TOGGLE("Enabled", "sub-visibility"),
        OPT("Load subtitles from file", activate=load_sub_file),
        OPT("Download subtitles",
            activate=post_menu_action_factory(dl_subs))
    ),
    SEP,
    Layout("Help", OPT("About", activate=about)),
)


class Menu(Gtk.Menu):
    def __init__(self, layout):
        Gtk.Menu.__init__(self)
        self.process_layout(layout)
        self.action = ""
        self.show_all()
        self.connect("selection-done",
                     self.on_selection_done)
        self.popup(None, None,
                   None, None,
                   2,
                   Gtk.get_current_event_time())
        self.activation_handled = False

    def on_selection_done(self, widget):
        Gtk.main_quit()
        return True

    def on_menu_item_activate(self, widget):
        global action
        if not self.activation_handled:
            widget.activate()
            self.activation_handled = True
        return True

    def on_menu_item_btn(self, widget, evt):
        if evt.button != 1:
            return False
        return self.on_menu_item_activate(widget)

    def process_layout(self, layout, menu=None):
        if not menu:
            menu = self
        for item in layout.items:
            if isinstance(item, Layout):
                menu_item = Gtk.MenuItem(item.name)
                submenu = Gtk.Menu()
                menu_item.set_submenu(submenu)
                menu.append(menu_item)
                self.process_layout(item, submenu)
            else:
                if item.typ == OPT.SEP:
                    menu_item = Gtk.SeparatorMenuItem()
                else:
                    item.init()
                    if item.typ == OPT.CHECK:
                        menu_item = Gtk.CheckMenuItem(item.name)
                        menu_item.set_active(item.state)
                    else:
                        menu_item = Gtk.MenuItem(item.name)
                    menu_item.activate = item.activate
                    menu_item.connect("activate",
                                      self.on_menu_item_activate)
                    # workaround for bug #695488
                    menu_item.connect("button-press-event",
                                      self.on_menu_item_btn)
                menu.append(menu_item)


def run_client(script_message):
    global rpc
    global post_menu_action
    rpc = RPC("/var/run/user/{}/mpv.sock".format(os.getuid()))

    wd = rpc.get_prop("working-directory")
    if wd:
        WORK_DIR = wd  # noqa
    else:
        LOGGER.warn("Can't get mpv's working directory,",
                    "using current working dir of this script.")

    while True:
        dat = rpc.recv_data()
        if not dat:
            break
        # dispatch
        if "event" in dat:
            # event
            if dat["event"] == "client-message" and \
               dat["args"][0] == script_message:
                menu = Menu(layout)  # noqa
                Gtk.main()
                if post_menu_action:
                    logging.debug("Calling post menu action.")
                    post_menu_action()
                    post_menu_action = None


if __name__ == "__main__":
    arg_parser = argparse.ArgumentParser(description="Show a menu" +
                                         " when an action occurs in MPV.")
    arg_parser.add_argument("--log-level", type=str, choices=["debug", "info",
                                                              "warn", "error"],
                            default="warn", help="logging level")
    arg_parser.add_argument("--script-message", type=str, default="popup_menu",
                            help="custom script message to trigger the menu")
    args = arg_parser.parse_args()

    logging.basicConfig(level=getattr(logging, args.log_level.upper()),
                        format="[%(asctime)s] %(levelname)s in " +
                        "%(funcName)s at %(lineno)d: %(message)s")

    signal.signal(signal.SIGINT, signal.SIG_DFL)
    while True:
        try:
            run_client(args.script_message)
        except (BrokenPipeError, ConnectionResetError):
            pass
