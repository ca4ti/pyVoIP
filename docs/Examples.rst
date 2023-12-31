Examples
########

Here we will go over a few basic phone setups.

Setup
*****

PyVoIP uses callback functions to initiate phone calls.  In the example below, our callback function is named ``answer``.  The callback takes one argument, which is a :ref:`VoIPCall` instance.

We are also importing :ref:`VoIPPhone` and :ref:`InvalidStateError<invalidstateerror>`.  VoIPPhone is the main class for our `softphone <https://en.wikipedia.org/wiki/Softphone>`_.  An InvalidStateError is thrown when you try to perform an impossible command.  For example, denying the call when the phone is already answered, answering when it's already answered, etc.

The following will create a phone that answers and automatically hangs up:

.. code-block:: python
   
  from pyVoIP.VoIP import VoIPPhone, InvalidStateError

  def answer(call):
      try:
          call.answer()
          call.hangup()
      except InvalidStateError:
          pass
  
  if __name__ == "__main__":
      phone = VoIPPhone(<SIP server IP>, <SIP server port>, <SIP server username>, <SIP server password>, myIP=<Your computer's local IP>, callCallback=answer)
      phone.start()
      input('Press enter to disable the phone')
      phone.stop()
    
Announcement Board
******************

Let's say you want to make a phone that when you call it, it plays an announcement message, then hangs up.  We can accomplish this with the builtin libraries `wave <https://docs.python.org/3/library/wave.html>`_, `audioop <https://docs.python.org/3/library/audioop.html>`_, `time <https://docs.python.org/3/library/time.html>`_, and by importing :ref:`CallState<callstate>`.

.. code-block:: python

  from pyVoIP.VoIP import VoIPPhone, InvalidStateError, CallState
  import time
  import wave
  
  def answer(call):
      try:
          f = wave.open('announcment.wav', 'rb')
          frames = f.getnframes()
          data = f.readframes(frames)
          f.close()
      
          call.answer()
          call.write_audio(data)  # This writes the audio data to the transmit buffer, this must be bytes.
      
          stop = time.time() + (frames / 8000)  # frames/8000 is the length of the audio in seconds. 8000 is the hertz of PCMU.
      
          while time.time() <= stop and call.state == CallState.ANSWERED:
              time.sleep(0.1)
          call.hangup()
      except InvalidStateError:
          pass
      except:
          call.hangup()
  
      
  if __name__ == "__main__":
      phone = VoIPPhone(<SIP Server IP>, <SIP Server Port>, <SIP Server Username>, <SIP Server Password>, myIP=<Your computers local IP>, callCallback=answer)
      phone.start()
      input('Press enter to disable the phone')
      phone.stop()

Something important to note is our wait function.  We are currently using:

.. code-block:: python

  stop = time.time() + (frames / 8000)  # The number of frames/8000 is the length of the audio in seconds.
      
  while time.time() <= stop and call.state == CallState.ANSWERED:
      time.sleep(0.1)

This could be replaced with ``time.sleep(frames / 8000)``.  However, doing so will not cause the thread to automatically close if the user hangs up, or if ``VoIPPhone().stop()`` is called; using the while loop method will fix this issue.  The ``time.sleep(0.1)`` inside the while loop is also important.  Supplementing ``time.sleep(0.1)`` for ``pass`` will cause your CPU to ramp up while running the loop, making the RTP (audio being sent out and received) lag.  This can make the voice audibly slow or choppy.

*Note: Audio must be 8 bit, 8000Hz, and Mono/1 channel.  You can accomplish this in a free program called* `Audacity <https://www.audacityteam.org/>`_.  *To make an audio recording Mono, go to Tracks > Mix > Mix Stereo Down to Mono.  To make an audio recording 8000 Hz, go to Tracks > Resample... and select 8000, then ensure that your 'Project Rate' in the bottom left is also set to 8000.  To make an audio recording 8 bit, go to File > Export > Export as WAV, then change 'Save as type:' to 'Other uncompressed files', then set 'Header:' to 'WAV (Microsoft)', then set the 'Encoding:' to 'Unsigned 8-bit PCM'*

IVR/Phone Menus
****************

We can use the following code to create `IVR Menus <https://en.wikipedia.org/wiki/Interactive_voice_response>`_.  Currently, we cannot make 'breaking' IVR menus.  Breaking IVR menus in this context means, a user selecting an option mid-prompt will cancel the prompt, and start the next action.  Support for breaking IVR's will be made in the future.  For now, here is the code for a non-breaking IVR:

.. code-block:: python

  from pyVoIP.VoIP import VoIPPhone, InvalidStateError, CallState
  import time
  import wave
  
  def answer(call):
      try:
          f = wave.open('prompt.wav', 'rb')
          frames = f.getnframes()
          data = f.readframes(frames)
          f.close()
      
          call.answer()
          call.write_audio(data)
      
          while call.state == CallState.ANSWERED:
              dtmf = call.get_dtmf()
              if dtmf == "1":
                  # Do something
                  call.hangup()
              elif dtmf == "2":
                  # Do something else
                  call.hangup()
              time.sleep(0.1)
      except InvalidStateError:
          pass
      except:
          call.hangup()
      
  if __name__ == '__main__':
      phone = VoIPPhone(<SIP Server IP>, <SIP Server Port>, <SIP Server Username>, <SIP Server Password>, myIP=<Your computers local IP>, callCallback=answer)
      phone.start()
      input('Press enter to disable the phone')
      phone.stop()

Please note that ``get_dtmf()`` is actually ``get_dtmf(length=1)``, and as it is technically an ``io.StringBuffer()``, it will return ``""`` instead of ``None``.  This may be important if you wanted an 'if anything else, do that' clause.  Lastly, VoIPCall stores all DTMF keys pressed since the call was established; meaning, users can press any key they want before the prompt even finishes, or may press a wrong key before the prompt even starts.

