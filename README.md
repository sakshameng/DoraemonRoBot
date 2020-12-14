#DoraemonRoBot
import importlib
import time
import re
from sys import argv
from typing import Optional

from SaitamaRobot import (ALLOW_EXCL, CERT_PATH, DONATION_LINK, LOGGER,
                          OWNER_ID, PORT, SUPPORT_CHAT, TOKEN, URL, WEBHOOK,
                          SUPPORT_CHAT, dispatcher, StartTime, telethn, updater)
# needed to dynamically load modules
# NOTE: Module order is not guaranteed, specify that in the config file!
from SaitamaRobot.modules import ALL_MODULES
from SaitamaRobot.modules.helper_funcs.chat_status import is_user_admin
from SaitamaRobot.modules.helper_funcs.misc import paginate_modules
from telegram import (InlineKeyboardButton, InlineKeyboardMarkup, ParseMode,
                      Update)
from telegram.error import (BadRequest, ChatMigrated, NetworkError,
                            TelegramError, TimedOut, Unauthorized)
from telegram.ext import (CallbackContext, CallbackQueryHandler, CommandHandler,
                          Filters, MessageHandler)
from telegram.ext.dispatcher import DispatcherHandlerStop, run_async
from telegram.utils.helpers import escape_markdown


def get_readable_time(seconds: int) -> str:
    count = 0
    ping_time = ""
    time_list = []
    time_suffix_list = ["s", "m", "h", "days"]

    while count < 4:
        count += 1
        if count < 3:
            remainder, result = divmod(seconds, 60)
        else:
            remainder, result = divmod(seconds, 24)
        if seconds == 0 and remainder == 0:
            break
        time_list.append(int(result))
        seconds = int(remainder)

    for x in range(len(time_list)):
        time_list[x] = str(time_list[x]) + time_suffix_list[x]
    if len(time_list) == 4:
        ping_time += time_list.pop() + ", "

    time_list.reverse()
    ping_time += ":".join(time_list)

    return ping_time


PM_START_TEXT = """
Hi {}, my name is {}! 
I am an Anime themed group management bot.
Build by weebs for weebs, I specialize in managing anime and similar themed groups.
You can find my list of available commands with /help.
"""

HELP_STRINGS = """
Hey there! My name is *{}*.
I'm a Hero For Fun and help admins manage their groups with One Punch! Have a look at the following for an idea of some of \
the things I can help you with.

*Main* commands available:
 • /help: PM's you this message.
 • /help <module name>: PM's you info about that module.
 • /donate: information on how to donate!
 • /settings:
   • in PM: will send you your settings for all supported modules.
   • in a group: will redirect you to pm, with all that chat's settings.


{}
And the following:
""".format(
    dispatcher.bot.first_name, ""
    if not ALLOW_EXCL else "\nAll commands can either be used with / or !.\n")

SAITAMA_IMG = ""

DONATE_STRING = """Heya, glad to hear you want to donate!
Saitama is hosted on one of Kaizoku's Servers and doesn't require any donations as of now but \
You can donate to the original writer of the Base code, Goku #Lover
There are two ways of supporting him; [Telegram](https:t.me/FaucetMaker)."""

IMPORTED = {}
MIGRATEABLE = []
HELPABLE = {}
STATS = []
USER_INFO = []
DATA_IMPORT = []
DATA_EXPORT = []
CHAT_SETTINGS = {}
USER_SETTINGS = {}

for module_name in ALL_MODULES:
    imported_module = importlib.import_module("SaitamaRobot.modules." +
                                              module_name)
    if not hasattr(imported_module, "mod_name"):
        imported_module.mod_name = imported_module.name

    if not imported_module.mod_name.lower() in IMPORTED:
        IMPORTED[imported_module.mod_name.lower()] = imported_module
    else:
        raise Exception(
            "Can't have two modules with the same name! Please change one")

    if hasattr(imported_module, "help") and imported_module.help:
        HELPABLE[imported_module.mod_name.lower()] = imported_module

    # Chats to migrate on chat_migrated events
    if hasattr(imported_module, "migrate"):
        MIGRATEABLE.append(imported_module)

    if hasattr(imported_module, "stats"):
        STATS.append(imported_module)
