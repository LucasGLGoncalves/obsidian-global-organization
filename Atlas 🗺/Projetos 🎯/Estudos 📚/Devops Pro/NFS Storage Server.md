### Primeiro eu devo instalar o NFS kernel server

sudo apt update && \  
sudo apt install nfs-kernel-server --yes

Crio o diretório que quero compartilhar

sudo mkdir -p /mnt/nfs_shared   
sudo chown -R nobody:nogroup /mnt/nfs_shared/  
sudo chmod 777 /mnt/nfs_shared/

Compartilhamento do diretório

Agora eu mapeio os diretório que vou utilizar

sudo vim /etc/exports

/mnt/nfs_shared *(rw,sync,no_subtree_check,no_root_squash,insecure)

- /mnt/nfs_shared: Caminho no servidor NFS que está sendo compartilhado. Representa a localização dos arquivos e diretórios disponíveis para os clientes NFS.  
- * : Indica que qualquer cliente pode montar este compartilhamento, representando todos os endereços IP. Para maior segurança, pode ser substituído por endereços IP específicos ou faixas de IP.  
- rw: Permissão de leitura e escrita para os clientes NFS, permitindo que modifiquem os arquivos no compartilhamento.  
- sync: As modificações nos arquivos são escritas no disco antes que as operações sejam consideradas concluídas, aumentando a integridade dos dados.  
- no_subtree_check: Desativa a verificação de sub-árvores para melhorar o desempenho do NFS, embora deva ser usada com cautela.  
- no_root_squash: Desativa a transformação de solicitações do usuário root em solicitações não privilegiadas, aumentando o risco de segurança mas permitindo que o root nos clientes NFS tenha os mesmos privilégios que o root no servidor.  
- insecure: Permite conexões de clientes NFS usando portas acima de 1024, necessário em alguns ambientes ou configurações de firewall.

Reinicio o serviço

sudo exportfs -a  
sudo systemctl restart nfs-kernel-server


#DevOps #Estudos #Tecnologia #Recorrente #Dicas #Comandos #Linux