```
void *new_thread(void *)
{
	sleep(1);
	system("ls");
}
void main()
{
	fd = open("/dev/ttyS0", O_RDONLY);
	pthread_create(new_thread);

	while (1) {
		ret = read(fd, buff);
		if (ret > 0) {
			printf("%s\n", buff);
		} else if (ret == 0) {
			printf("read nothing\n");
		} else if (ret < 0) {
			printf("read error");
			exit(-1);
		}
	}
}
```
