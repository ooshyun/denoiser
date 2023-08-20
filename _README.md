# Analyze Denoiser

## Params

- model.sample_rate = 16000
- time per frame (tpf) = time_per_frame * 1000
- Real Time Factor(RTF) = tpf / stride_ms
- total lag = total_length / sr_ms
- last_error_time, cooldown_time: overflow / underflow 시 tpf log
- last_log_time, log_delta : 지난 시간

- self.total_length = self.frame_length + self.resample_lookahead
- self.stride = demucs.total_stride * num_frame
- overflow: ``True`` if input data was discarded by PortAudio after the previous call and before this call.
- underflow: ``True`` "audio dropouts", if additional output data was inserted after the previous call and before this call.

## Sequence

1. if 현재 시간 > last_log_time + log_delta:
    - log tpf, rtf > reset

2. frame, overflow = stream_in

3. mean in channel

4. feed to streamer

5. appending pending buffer
    ```markdown
                 661(total_length)
         _____________________________________
        |                    |   256(stride)  |
        |____________________|________________|
    
    ```
6. while pending length >= total length
    1. frame = pending[:total_length]
       - dry signal = frame [:stride]
    
    2. normalize frame
        - [TODO]

    3. padded_frame: append to frame(661) using resample-in(256)
    ```markdown
            917 (661(total_length) + 256(resample-in))
         _____________________________________________________
        |  resample in (256) |           frame (661)          |
        |____________________|________________________________|
    ```
    4. get resample-in(256)
    ```markdown
               917 (661(total_length) + 256(resample-in))
         ____________________________________________________
        |  | 256(resample buffer) |                          |
        |__|______________________|__________________________|
           ^                  ^ 
    stride - resample buffer  |
                            stride
    ```        
    5. fir resample (none / x2 / x4 )
        - in example, x4: 917 * 4 = 3668

    6. remove pre sampling buffer (3668 - 256*4 = 2644)
    
    ```markdown
                        917 * resample
         ________________________________________________
        | remove |                                      |
        |________|______________________________________|
                 ^
       resample * resample buffer
    ```

    7. remove extar samples after window (2644 - 597 = 2388)
        - (597) frame_length = demucs.valid_length(1) + demucs.total_stride * (num_frames - 1)
        - (597) demucs.valid_length(1) = 597
            - If the mixture has a valid length, the estimated sources
        will have exactly the same length.
        - (256) demucs.total_stride = self.stride ** self.depth // self.resample
        - num_frames = 1
            - num_frames (int): number of frames to process at once. Higher values
            will increase overall latency but improve the real time factor.
        ```markdown
         ___________________________________________
        |    resample * frame_length    |  remove   |                          
        |_______________________________|___________|
                                        ^
                               resample * frame_length
        ```
    
    8. Separate Frame (Demuc)
        - [TODO]

    9. Padding Output

        ```markdown
         ________________________________________________________________________________________
        |    256 (resample_out)  |           1024(out)            |         1364(extra)          |
        |________________________|________________________________|______________________________|

        ```
    
    10. get resample_out
        
        ```markdown
                                                                            256 (resample_out)
         ________________________________________________________________________________________
        |                                                               |                        |
        |_______________________________________________________________|________________________|

        ```    

    11. down sample (none / x2 / x4 ) 
        - In an example, x4: 2664 > 661
    
    12. remove prepadding ( 661 - 256//4 = 597)

       ```markdown
         ___________________________________________
        | remove |                                 |                          
        |________|_________________________________|
                 ^
     resample_buffer // resample
        ```

    13. getting front of out (597 > 256)
        
        ```markdown
                            out
         ___________________________________________
        |  out   |                                 |                          
        |________|_________________________________|
                 ^
               stride
        ```

    14. Denormalize frame
        - [TODO]

    15. adjust original signal
        - out = self.dry * dry_signal + (1 - self.dry) * out

    16. Get pending
        ```markdown
                            pending
         ___________________________________________
        | remove |                                 |                          
        |________|_________________________________|
                 ^
               stride
        ```

7. if compressor: out = 0.99 * torch.tanh(out)
8. out.clamp_(-1, 1)
9. stream_out