#define _GNU_SOURCE
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

static int connect_to(char *ip, int port)
{
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        printf("Failed to create socket.\n");
        return -1;
    }

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));

    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);

    if (inet_pton(AF_INET, ip, &addr.sin_addr) <= 0) {
        perror("inet_pton");
        return -1;
    }

    if (connect(sockfd, (struct sockaddr *) &addr, sizeof(addr)) < 0) {
        printf("Failed to connect.\n");
        return -1;
    }

    return sockfd;
}

static void move(int in_fd, int out_fd, int pip[2])
{
    int ret = splice(in_fd, NULL, pip[1], NULL, 8, SPLICE_F_MOVE);
    if (ret == -1) {
        perror("splice(1)");
        return;
    }

    ret = splice(pip[0], NULL, out_fd, NULL, 8, SPLICE_F_MOVE);
    if (ret == -1) {
        perror("splice(1)");
        return;
    }
}

#define MAX_EVENTS 10
struct epoll_event ev, events[MAX_EVENTS];

static void proxy(int cl_fd, int target_fd)
{
    if (target_fd == -1)
        return;

    int fds[2];
    if (pipe(fds) == -1) {
        perror("pipe");
        return;
    }
    
    epollfd = epoll_create1(0);
    if (epollfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }
    
    ev.events = EPOLLIN;
    ev.data.fd = cl_fd;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, cl_fd, &ev) == -1) {
        perror("epoll_ctl: cl_fd");
        exit(EXIT_FAILURE);
    }    

    ev.events = EPOLLIN;
    ev.data.fd = target_fd;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, target_fd, &ev) == -1) {
        perror("epoll_ctl: target_fd");
        exit(EXIT_FAILURE);
    }  

    int ret;
    while ((ret = epoll_wait(epollfd, events, MAX_EVENTS, 1000)) != -1) {
        if (ret == -1) {
            perror("epoll_wait");
            exit(EXIT_FAILURE);
        }
        if (ret == 0)
            continue;
            
        for (int i = 0; i < ret; ++i) {
            int from_client = (events[i].events & POLLIN) && (events[i].data.fd == cl_fd);
            if (from_client)
                move(cl_fd, target_fd, fds);
            else
                move(target_fd, cl_fd, fds);
        }
    }

    perror("epoll");
}

#define PORT 1922

int main(int argc, char *argv[])
{
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <target IP address> <target port>\n",
                argv[0]);
        return -1;
    }

    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));

    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(PORT);

    int optval = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof(optval));

    bind(listenfd, (struct sockaddr *) &addr, sizeof(addr));

    listen(listenfd, 1);

    while (1) {
        int connfd = accept(listenfd, (struct sockaddr *) NULL, NULL);
        int target_fd = connect_to(argv[1], atoi(argv[2]));
        if (target_fd >= 0)
            proxy(connfd, target_fd);

        close(connfd);
    }
    return 0; /* should not reach here */
}
