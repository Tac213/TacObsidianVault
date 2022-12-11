使用下面的方法，既可以获取输出，又可以实时打印log:

```python
import os
import threading


class LogPipe(threading.Thread):

    def __init__(self) -> None:
        super().__init__()
        self.daemon = False
        self.fd_read, self.fd_write = os.pipe()
        self.pipe_reader = os.fdopen(self.fd_read)
        self.start()

    def test_method(self):
        pass

    def fileno(self) -> int:
        """
        Return the write file descriptor of the pipe
        """
        return self.fd_write

    def run(self):
        """
        Run the thread, logging everything.
        """
        for line in iter(self.pipe_reader.readline, ''):
            print(line.strip('\n'))
        self.pipe_reader.close()

    def close(self):
        """
        Close the write end of the pipe.
        """
        os.close(self.fd_write)


if __name__ == '__main__':
    import subprocess
    print('start')
    stdout = LogPipe()
    stderr = LogPipe()
    with subprocess.Popen('ping www.google.com', stdout=stdout, stderr=stderr, shell=True) as p:
        p.wait()
        stdout.close()
        stderr.close()
    print('end')
```
