# lex-bot-tester
**AWS Lex Bot Tester** is a library that simplifies the creation of AWS Lex Bot tests.

Using AWS Lex Models Client this utility inspects the properties of the available Bots and creates specific Results classes to be used by the tests.

Certainly, there are ways of testing your bots using AWS CLI as explained in [Test the Bot Using Text Input (AWS CLI)](http://docs.aws.amazon.com/lex/latest/dg/gs-create-test-text.html) but **lex-bot-tester** provides a more concise, type safe and object oriented way of doing it.

# Installation
Run

    pip install lex-bot-tester
    

# Example
You may be familiar with this kind of tests in the *AWS Lex Console* (this example uses the well know *OrderFlowers* bot).

![alt text](https://raw.githubusercontent.com/dtmilano/lex-bot-tester/master/images/test-bot.png)

> More information about these manual tests using the console can be found [here](http://docs.aws.amazon.com/lex/latest/dg/gs2-build-and-test.html)

However, once you have the **lex-bot-tester** installed, you can create tests like this one:

```python
#! /usr/bin/env python
# -*- coding: utf-8 -*-
"""
    Lex Bot Tester
    Copyright (C) 2017  Diego Torres Milano

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.
"""
import re
import unittest

from com.dtmilano.aws.lex.conversation import Conversation, ConversationItem
from com.dtmilano.aws.lex.lexbottest import LexBotTest
from com.dtmilano.aws.lex.lexmodelsclient import LexModelsClient
from com.dtmilano.aws.lex.lexruntimeclient import DialogState

RE_DATE = re.compile('\d+-\d+-\d+')

BOT_NAME = 'OrderFlowers'
BOT_ALIAS = 'OrderFlowersLatest'
USER_ID = 'ClientId'


class OrderFlowersTests(LexBotTest):
    def test_conversations_text(self):
        lmc = LexModelsClient(BOT_NAME, BOT_ALIAS)
        conversations = []
        for i in lmc.get_intents_for_bot():
            r = lmc.get_result_class_for_intent(i)
            if i == 'OrderFlowers':
                conversations.append(Conversation(
                    ConversationItem('I would like to order some roses',
                                     r(DialogState.ELICIT_SLOT, flower_type='roses')),
                    ConversationItem('white', r(DialogState.ELICIT_SLOT, flower_type='roses', flower_color='white')),
                    ConversationItem('next Sunday',
                                     r(DialogState.ELICIT_SLOT, flower_type='roses', flower_color='white',
                                       pickup_date=RE_DATE)),
                    ConversationItem('noon', r(DialogState.CONFIRM_INTENT, flower_type='roses', flower_color='white',
                                               pickup_date=RE_DATE, pickup_time='12:00')),
                    ConversationItem('yes', r(DialogState.FULFILLED, flower_type='roses', flower_color='white',
                                              pickup_date=RE_DATE, pickup_time='12:00')),
                ))
            elif i == 'Cancel':
                conversations.append(Conversation(
                    ConversationItem('Cancel', r(DialogState.READY_FOR_FULFILLMENT))
                ))
        self.conversations_text(BOT_NAME, BOT_ALIAS, USER_ID, conversations)


if __name__ == '__main__':
    unittest.main()
```

This test, first creates a `LexModelsClient` to inspect the definitions of the bot, its intents and slots to later use a class factory that defines specific classes for each intent which are obtained by `get_result_class_for_intent(i)`.

This result class reference, which extends `ResultBase` class is assigned to the variable `r` for convenience. Then, for each intent, a `Conversation`, consisting of a list of `ConversationItems` is created.

`ConversationItem` specifies the text or utterance sent and the expected result, using the `r` class reference and invoking the constructor with the expected `DialogState` and the values of the `slots`.

`pickup_date` is a particular case, as it is selected as `next Sunday` so instead of looking for a particular value we are checking if it matches a regular expression defining dates.

Finaly, once the `conversation` list is completed, a call to the helper method `conversations_text` providing this list as an argument completes the test.

# Result classes
As mentioned before, `LexModelsClient.get_result_class_for_intent(intent)` returns the class that represents the response result once the Bot is invoked using the corresponding utterance.

The signature of the constructor matches this pattern

    class MyIntentResult(ResultBase):
        def __init__(dialog_state, **kwargs):
            ...
            

To comply with [PEP 8](https://www.python.org/dev/peps/pep-0008/#prescriptive-naming-conventions), keyword args representing slots are named using *snake case* when usually slots are named using *camel case*. Then, for example, the slot `FlowerType` will be represented by its corresponding keyword arg `flower_type`.
