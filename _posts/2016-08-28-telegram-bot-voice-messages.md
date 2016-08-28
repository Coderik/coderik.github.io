---
layout: post
title:  "Voice messages with python-telegram-bot"
date:   2016-08-28 11:10:00 +0200
categories: telegram python
---
Recently I was playing around with the Telegram's Bot API using the [python-telegram-bot](https://github.com/python-telegram-bot/python-telegram-bot) library. I have faced a small but curious issue. Let's say that we want to sent an mp3 file to a Telegram chat. We have two options here: either send the file as a voice message or as a general file. In the first case the audio should be playable right through any Telegram client, while in the second case a user should be able to download it. With python-telegram-bot this can be done via methods Bot.sendVoice and Bot.sendDocument.

By the way, for the web client both options seem to be equal at a first sight: a user has *Download* and *Play* links in either case. However, when we send an audio as a voice message, it gets wrapped in the mpeg container, and that is what our user downloads.

<!--more-->

Let me show you the following signatures:

{% highlight python %}
# first
def sendVoice(self, chat_id, voice, duration=None, **kwargs):
# second
def sendDocument(self, chat_id, document, filename=None, 
caption=None, **kwargs):
{% endhighlight %}

The signatures are quite different, but pay special attention to the lack of *filename* parameter in sendVoice. And no, you cannot pass it in kwargs.

Now, let's say that we have an audio in a memory buffer and we want to send it as a voice message. For the sake of example we can do something like this:

{% highlight python %}
import requests
import telegram
from io import BytesIO

# Parameters
bot_token = 'your-bot-token'
chat_id = 12345678
song_url = 'http://someserver/song.mp3'

bot = telegram.Bot(token=bot_token)
song_response = requests.get(song_url)
if song_response.status_code == 200:
	voice = BytesIO(song_response.content)
	bot.sendVoice(chat_id, voice=voice)
{% endhighlight %}

Of course, if we just want to resend an audio file from another server, we can simply pass its url to sendVoice:

{% highlight python %}
bot.sendVoice(chat_id, voice=song_url)
{% endhighlight %}

This would do the trick, but we make it the long way only to have our audio as an array of bytes in memory. And here is where the problem arises. We get "TypeError: expected string or buffer" from inside urllib. Let's see what is happening here.

First of all we get into *sendVoice*:

{% highlight python %}
@message
def sendVoice(self, chat_id, voice, duration=None, **kwargs):
    url = '{0}/sendVoice'.format(self.base_url)
    data = {'chat_id': chat_id, 'voice': voice}

    if duration:
        data['duration'] = duration

    return url, data
{% endhighlight %}

Whole message-sending routine is inside *message* decorator:

{% highlight python %}
def message(func):

    @functools.wraps(func)
    def decorator(self, *args, **kwargs):
        url, data = func(self, *args, **kwargs)

        # kwargs processing is skipped here

        result = request.post(url, data, 
        	timeout=kwargs.get('timeout'))

        # result processing is skipped here

    return decorator
{% endhighlight %}

So, *sendVoice* returns *data* dictionary that contains our message and some metadata, then *message* decorator sends this data. I have skipped kwargs processing, but I can assure you that it is irrelevant for now. Let's go deeper:

{% highlight python %}
def post(url, data, timeout=None):
    if InputFile.is_inputfile(data):
        data = InputFile(data)
        result = _request_wrapper('POST', url, 
        	body=data.to_form(), headers=data.headers)
    else:
        # posting data as json is skipped here

    return _parse(result)
{% endhighlight %}

Method InputFile.is_inputfile() checks that *data* represents some file and not a text message in json formant. It returns True for 'audio', 'document', 'photo', 'sticker', 'video', 'voice' and 'certificate' types of messages. After the check is passed, *data* is wrapped into *InputFile*.

{% highlight python %}
class InputFile(object):
def __init__(self, data):
    self.data = data

    # ...
    if 'voice' in data:
        self.input_name = 'voice'
        self.input_file = data.pop('voice')
    # ...

    if hasattr(self.input_file, 'read'):
        self.filename = None
        self.input_file_content = self.input_file.read()
        if 'filename' in data:
            self.filename = self.data.pop('filename')
        elif hasattr(self.input_file, 'name'):
            self.filename = 
            	os.path.basename(self.input_file.name)

        try:
            self.mimetype = 
            	InputFile.is_image(self.input_file_content)
            if not self.filename or '.' not in self.filename:
                self.filename = self.mimetype.replace('/','.')
        except TelegramError:
            self.mimetype = mimetypes.guess_type(self.filename)[0] or DEFAULT_MIME_TYPE
{% endhighlight %}

Recall that we are trying to send an in-memory buffer as a voice message, so I have removed everything related to resending an audio from some given url. Since we are dealing with some kind of a buffer, hasattr(self.input_file, 'read') returns True. But what happens next? We are defining *filename*! First we look for it in *data* dictionary, but it is not there, because *sendVoice* does not allow us to specify it. Then we check that maybe our buffer (BytesIO) itself has *name* attribute, but it does not (which makes sense). And we end up with mime type guessing which throws TypeError, even though we know for sure that we are trying to send an mp3 file. We just need a way to communicate the filename.

Notice that the code above works perfectly fine when we send a file from the disk.

{% highlight python %}
bot.sendVoice(chat_id, voice=open('/path/to/song.mp3', 'rb'))
{% endhighlight %}

This is because type 'file', returned by open(), has *name* attribute.

How can we communicate the filename without altering the code of the library? An obvious workaround is to add the notorious *name* attribute to our buffer:

{% highlight python %}
class BufferWithName(BytesIO):
	def __init__(self, buffer=None, name=''):
		super(BufferWithName, self).__init__(buffer)
		self.name = name
{% endhighlight %}

Then we just need to use it instead of BytesIO:

{% highlight python %}
song_filename = somehow_extract_name_from_url(song_url)
voice = BufferWithName(song_response.content, song_filename)
bot.sendVoice(chat_id, voice=voice)
{% endhighlight %}
