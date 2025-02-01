**Oi, eu sou g4bbdev.**  
Tenho 13 anos e ,  

Há três meses, descobri um ataque de desanonimização "0-click" que permite a um invasor obter a localização de qualquer alvo dentro de um raio de 400 km. Se a vítima tiver um aplicativo vulnerável instalado no celular (ou um programa em execução em segundo plano no laptop), um atacante pode enviar uma carga maliciosa e rastrear sua localização em segundos—sem que você perceba.  

Estou publicando esta pesquisa como um alerta, especialmente para jornalistas, ativistas e hackers, sobre esse tipo de ataque indetectável. Centenas de aplicativos estão vulneráveis, incluindo alguns dos mais populares do mundo: Signal, Discord, Twitter/X e outros. Aqui está como funciona:  

## Cloudflare  

Em termos de números, a Cloudflare é facilmente a CDN mais popular do mercado, superando concorrentes como Sucuri, Amazon CloudFront, Akamai e Fastly. Em 2019, uma grande falha na Cloudflare tirou do ar boa parte da internet por mais de 30 minutos.  

Um dos recursos mais usados da Cloudflare é o **Cache**. A Cloudflare armazena cópias de conteúdos frequentemente acessados (como imagens, vídeos ou páginas da web) em seus data centers, reduzindo a carga nos servidores e melhorando o desempenho dos sites ([documentação oficial](https://developers.cloudflare.com/cache/)).  

Quando seu dispositivo solicita um recurso que pode ser armazenado em cache, a Cloudflare primeiro tenta buscá-lo no data center mais próximo. Se não estiver disponível, o recurso é carregado do servidor de origem, armazenado localmente e entregue ao usuário. Por padrão, [algumas extensões de arquivos](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) são automaticamente armazenadas, mas os operadores do site podem definir novas regras de cache.  

A Cloudflare tem uma presença global massiva, com data centers em mais de **330 cidades e 120 países**—273% a mais que o Google. Nos EUA, por exemplo, o data center mais próximo de mim está a menos de 160 km. Se você mora em um país desenvolvido, é provável que haja um data center Cloudflare a menos de **320 km** de distância.  

Foi então que tive um momento de "eureka": **se a Cloudflare armazena dados em cache tão perto dos usuários, será que isso poderia ser explorado para ataques de desanonimização em sites que não controlamos?**  

A resposta está nos cabeçalhos HTTP da Cloudflare:  
![cf-cache-status](https://gist.github.com/user-attachments/assets/95e1a39a-ed25-4531-9c57-a1b43c616519)  

- cf-cache-status pode indicar HIT/MISS (se um arquivo foi entregue do cache).  
- cf-ray contém o **código do aeroporto** mais próximo do data center que atendeu à solicitação.  

Se conseguirmos fazer o dispositivo da vítima carregar um recurso hospedado em um site que usa Cloudflare, podemos então **mapear todos os data centers** da Cloudflare para descobrir qual deles armazenou esse recurso. Isso nos dá uma estimativa extremamente precisa da localização da vítima.  

## Cloudflare Teleport  

Havia um grande obstáculo antes que eu pudesse testar essa teoria:  

**Não dá para enviar solicitações HTTP diretamente para data centers específicos da Cloudflare.** Toda a rede opera com **anycast**, ou seja, qualquer conexão TCP sempre é direcionada ao data center mais próximo disponível.  

Porém, depois de algumas pesquisas, encontrei [um post no fórum da Cloudflare](https://community.cloudflare.com/t/how-to-run-workers-on-specific-datacenter-colos/385851) que explicava um **bug** para contornar essa limitação com o Cloudflare Workers.  

Basicamente, descobri que usando um intervalo de IPs internos do **Cloudflare WARP (VPN da Cloudflare)**, era possível **forçar** certas solicitações a serem atendidas por um data center específico.  

Com base nisso, desenvolvi o **Cloudflare Teleport** ([GitHub](https://github.com/hackermondev/cf-teleport)), um proxy baseado em Cloudflare Workers que permite redirecionar requisições HTTP para data centers específicos. Por exemplo:  

🔹 https://cfteleport.xyz/?proxy=https://cloudflare.com/cdn-cgi/trace&colo=SEA  

Isso direciona a solicitação especificamente para o **data center de Seattle (SEA)**.  

Meses depois, a Cloudflare corrigiu esse bug, tornando essa ferramenta obsoleta, mas até então **ela funcionava perfeitamente para meus testes**.  

---

## Primeiro Ataque de Desanonimização  

Com o **Cloudflare Teleport**, pude testar minha teoria.  

Criei um pequeno programa CLI que fazia uma solicitação HTTP para um URL específico e listava **todos os data centers que armazenaram o cache do recurso**.  

Para o primeiro teste, escolhi o favicon da **Namecheap**:  

🔗 https://www.namecheap.com/favicon.ico  

Este era um recurso simples, estático e armazenado em cache pela Cloudflare.  

![cache hit](https://gist.github.com/user-attachments/assets/8da57801-ae8e-4adf-9a2e-ec6feec6086f)  

💥 Funcionou! Consegui ver **todos os data centers** que armazenaram o favicon da Namecheap nos últimos 5 minutos.  

Isso provou que era possível **usar o cache da Cloudflare para rastrear usuários e estimar suas localizações**.  

---

## Aplicação Real: Signal  

O **Signal**, um dos apps de mensagens mais seguros do mundo, **estava vulnerável** a esse ataque.  

### Ataque 1-Click  

Quando um usuário envia um **anexo** (ex.: imagem) no Signal, o arquivo é enviado para o CDN da Cloudflare (cdn2.signal.org).  

Assim que o destinatário **abre a conversa**, seu dispositivo baixa automaticamente o anexo, e a Cloudflare o armazena em cache.  

🔹 **Solução:** Bloqueei a solicitação HTTP ao CDN e enviei um anexo de teste. Depois, usei meu programa para mapear os data centers onde o arquivo foi armazenado.  

📍 O resultado? Descobri que **meu data center mais próximo era o de Newark, NJ (EWR)**, a **240 km da minha localização real**.  

---

## Ataque 0-Click  

Agora, **como executar esse ataque sem a vítima abrir o chat?**  

O segredo: **notificações push**.  

📌 No Signal, quando um usuário recebe uma mensagem com um anexo, o app **baixa automaticamente a imagem** para exibi-la na notificação push.  

Isso significa que **não é necessário nenhuma interação do usuário**. Só precisamos enviar uma imagem para a vítima e aguardar a notificação ser entregue!  

💥 Resultado: **a vítima é rastreada sem nem abrir o app!**  

---

## Aplicação Real: Discord  

O **Discord** também é vulnerável ao mesmo ataque. Mas, em vez de anexos, podemos explorar **avatares de perfil**.  

🔹 Quando alguém recebe um **pedido de amizade**, o avatar do remetente é carregado automaticamente **para ser exibido na notificação push**.  

Se trocarmos nosso avatar antes do ataque, garantimos que **ninguém mais tenha carregado esse arquivo**, tornando a detecção precisa.  

Criei um bot chamado **GeoGuesser**, que automatiza todo o processo e executa o ataque em **segundos**.  

---

## Conclusão  

- **Signal e Discord foram alertados, mas não corrigiram o problema.**  
- **Cloudflare corrigiu a falha que permitia acessar data centers específicos, mas o ataque ainda é possível de outras formas.**  

Este ataque mostra que mesmo apps que prezam pela **privacidade** podem **vazar informações sensíveis sem que o usuário perceba**. 🚨