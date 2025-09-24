Um código básico, há um loop contínuo, onde ele fica monitorando o perfil do Instagram. A cada intervalo definido, ele recarrega a página, pega os dados atuais do perfil (foto, bio, quantidade de posts, seguidores e seguindo), e compara com os dados que foram salvos na última verificação.

tendo mudanças, ele mostra no console de forma organizada, em tabela, o que mudou: o valor antigo e o valor novo de cada campo. Além disso, se a foto do perfil tiver mudado de verdade, ele faz o download dessa nova imagem para uma pasta específica.

E com isso, o código atualiza o arquivo JSON que guarda o estado atual do perfil, para que na próxima verificação ele possa comparar novamente. Se não houver nenhuma mudança, ele só avisa que nada foi alterado e continua monitorando.

Lembrando de que, antes de rodar o script, você precisa salvar a sessão logada no Playwright, loga uma vez manualmente e exportar o storage_state para esse arquivo.

Depois, toda vez que o script abre o navegador, ele usa essa sessão salva para acessar perfis que precisam de login.
