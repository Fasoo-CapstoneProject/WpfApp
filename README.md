# WpfApp

agent.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <winsock2.h>
#include <windows.h> 

#pragma comment(lib, "Ws2_32.lib")

#define PORT 8080
#define BUFFER_SIZE 1024

void handle_directory_request(const char *directory_path, SOCKET client_socket) {
	WIN32_FIND_DATA findFileData;
	HANDLE hFind;

	char search_path[BUFFER_SIZE];
	snprintf(search_path, sizeof(search_path), "%s\\*", directory_path);

	hFind = FindFirstFile(search_path, &findFileData);
	if (hFind == INVALID_HANDLE_VALUE) {
		const char *error_response = "Invalid directory path or access denied.";
		send(client_socket, error_response, strlen(error_response), 0);
		return;
	}

	char response[BUFFER_SIZE] = "Directory contents:\n";
	do {
		strcat_s(response, BUFFER_SIZE, findFileData.cFileName);
		strcat_s(response, BUFFER_SIZE, "\n");
	} while (FindNextFile(hFind, &findFileData) != 0);

	FindClose(hFind);
	send(client_socket, response, strlen(response), 0);
}

void handle_client(SOCKET client_socket) {
	char buffer[BUFFER_SIZE];
	int bytes_received = recv(client_socket, buffer, BUFFER_SIZE, 0);

	if (bytes_received == SOCKET_ERROR) {
		printf("Error receiving data: %d\n", WSAGetLastError());
		closesocket(client_socket);
		return;
	}

	buffer[bytes_received] = '\0';
	printf("Received data: %s\n", buffer);

	if (strstr(buffer, "GET_DIRECTORY")) {
		char directory_path[BUFFER_SIZE];
		sscanf_s(buffer, "GET_DIRECTORY|PATH=%s", directory_path, (unsigned)_countof(directory_path));

		handle_directory_request(directory_path, client_socket);
	}
	else if (strstr(buffer, "GET_FILES")) {
		const char *response = "Server acknowledges file request";
		send(client_socket, response, strlen(response), 0);
	}
	else {
		const char *response = "Unknown command";
		send(client_socket, response, strlen(response), 0);
	}

	closesocket(client_socket);
}

int main() {
	WSADATA wsaData;
	SOCKET server_socket, client_socket;
	struct sockaddr_in server_addr, client_addr;
	int addr_len = sizeof(client_addr);

	if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
		printf("WSAStartup failed: %d\n", WSAGetLastError());
		return EXIT_FAILURE;
	}

	server_socket = socket(AF_INET, SOCK_STREAM, 0);
	if (server_socket == INVALID_SOCKET) {
		printf("Socket creation failed: %d\n", WSAGetLastError());
		WSACleanup();
		return EXIT_FAILURE;
	}

	server_addr.sin_family = AF_INET;
	server_addr.sin_addr.s_addr = INADDR_ANY;
	server_addr.sin_port = htons(PORT);

	if (bind(server_socket, (struct sockaddr *)&server_addr, sizeof(server_addr)) == SOCKET_ERROR) {
		printf("Bind failed: %d\n", WSAGetLastError());
		closesocket(server_socket);
		WSACleanup();
		return EXIT_FAILURE;
	}

	if (listen(server_socket, 5) == SOCKET_ERROR) {
		printf("Listen failed: %d\n", WSAGetLastError());
		closesocket(server_socket);
		WSACleanup();
		return EXIT_FAILURE;
	}

	printf("Server listening on port %d...\n", PORT);

	while (1) {
		client_socket = accept(server_socket, (struct sockaddr *)&client_addr, &addr_len);
		if (client_socket == INVALID_SOCKET) {
			printf("Accept failed: %d\n", WSAGetLastError());
			continue;
		}

		handle_client(client_socket);
	}

	closesocket(server_socket);
	WSACleanup();
	return 0;
}
