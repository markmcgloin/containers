---

copyright:
  years: 2014, 2018
lastupdated: "2018-10-25"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:pre: .pre}
{:table: .aria-labeledby="caption"}
{:codeblock: .codeblock}
{:tip: .tip}
{:download: .download}



# Expondo apps com balanceadores de carga
{: #loadbalancer}

Exponha uma porta e use um endereço IP móvel para um balanceador de carga da Camada 4 para acessar um app conteinerizado.
{:shortdesc}



## Componentes e arquitetura do balanceador de carga
{: #planning}

Ao criar um cluster padrão, o {{site.data.keyword.containerlong_notm}} provisiona automaticamente uma sub-rede pública móvel e uma sub-rede privada móvel.

* A sub-rede pública móvel fornece 1 endereço IP público móvel que é usado pelo [ALB do Ingresso público](cs_ingress.html) padrão. Os outros 4 endereços IP públicos móveis podem ser usados para expor apps únicos para a Internet, criando um serviço de balanceador de carga público.
* A sub-rede privada móvel fornece 1 endereço IP privado móvel que é usado pelo [ALB do Ingresso privado](cs_ingress.html#private_ingress) padrão. Os outros 4 endereços IP privados móveis podem ser usados para expor apps únicos para uma rede privada, criando um serviço de balanceador de carga privado.

Os endereços IP públicos e privados móveis são estáticos e não mudam quando um nó do trabalhador é removido. Se o nó do trabalhador no qual o endereço IP do balanceador de carga está for removido, um daemon Keepalived que monitora constantemente o IP moverá automaticamente o IP para outro nó do trabalhador. É possível designar qualquer porta ao balanceador de carga e não ser ligado a um determinado intervalo de portas.

Um serviço de balanceador de carga também torna seu app disponível nos NodePorts do serviço. [NodePorts](cs_nodeport.html) são acessíveis em cada endereço IP público e privado para cada nó dentro do cluster. Para bloquear o tráfego para NodePorts enquanto você está usando um serviço de balanceador de carga, consulte [Controlando o tráfego de entrada para os serviços de balanceador de carga ou do NodePort](cs_network_policy.html#block_ingress).

O serviço de balanceador de carga serve como o ponto de entrada externo para solicitações recebidas para o app. Para acessar o serviço de balanceador de carga por meio da Internet, use o endereço IP público de seu balanceador de carga e a porta designada no formato `<IP_address>:<port>`. O diagrama a seguir mostra como um balanceador de carga direciona a comunicação da Internet para um app.

<img src="images/cs_loadbalancer_planning.png" width="550" alt="Exponha um app no {{site.data.keyword.containerlong_notm}} usando um balanceador de carga" style="width:550px; border-style: none"/>

1. Uma solicitação para seu app usa o endereço IP público de seu balanceador de carga e a porta designada no nó do trabalhador.

2. A solicitação é encaminhada automaticamente para o endereço IP e porta do cluster interno do serviço de balanceador de carga. O endereço IP do cluster interno é acessível somente dentro do cluster.

3. `kube-proxy` roteia a solicitação para o serviço de balanceador de carga do Kubernetes para o app.

4. A solicitação é encaminhada para o endereço IP privado do pod de app. O endereço IP de origem do pacote de solicitação é mudado para o endereço IP público do nó do trabalhador no qual o pod de app está em execução. Se múltiplas instâncias do app são implementadas no cluster, o balanceador de carga roteia as solicitações entre os pods de app.

** Clusters Multizone **:

Se você tiver um cluster de múltiplas zonas, as instâncias do app serão implementadas em pods em trabalhadores nas diferentes zonas. O diagrama a seguir mostra como um balanceador de carga clássico direciona a comunicação da Internet para um app em um cluster de múltiplas zonas.

<img src="images/cs_loadbalancer_planning_multizone.png" width="500" alt="Use um serviço de balanceador de carga para o balanceamento de carga de apps em clusters de múltiplas zonas" style="width:500px; border-style: none"/>

Por padrão, cada balanceador de carga é configurado somente em uma zona. Para alcançar alta disponibilidade, deve-se implementar um balanceador de carga em cada zona na qual você tem instâncias do app. As solicitações são manipuladas pelos balanceadores de carga em várias zonas em um ciclo round-robin. Além disso, cada balanceador de carga roteia solicitações para as instâncias do app em sua própria zona e para instâncias do app em outras zonas.

<br />


## Ativando o acesso público ou privado a um app em um cluster de múltiplas zonas
{: #multi_zone_config}

Nota:
  * Este recurso está disponível somente para clusters padrão.
  * Os serviços do balanceador de carga não suportam finalização do TLS. Se seu app requerer a finalização do TLS, será possível expor seu app usando [Ingresso](cs_ingress.html) ou configurar seu app para gerenciar a finalização do TLS.

Antes de iniciar:
  * Um serviço de balanceador de carga com um endereço IP privado móvel ainda tem um NodePort público aberto em cada nó do trabalhador. Para incluir uma política de rede para evitar o tráfego público, consulte [Bloqueando tráfego recebido](cs_network_policy.html#block_ingress).
  * Deve-se implementar um balanceador de carga em cada zona e cada balanceador de carga é designado a seu próprio endereço IP nessa zona. Para criar balanceadores de carga públicos, pelo menos uma VLAN pública deverá ter sub-redes móveis disponíveis em cada zona. Para incluir serviços de balanceador de carga privado, pelo menos uma VLAN privada deve ter sub-redes móveis disponíveis em cada zona. Para incluir sub-redes, consulte [Configurando sub-redes para clusters](cs_subnets.html).
  * Se você restringir o tráfego de rede aos nós do trabalhador de borda, assegure-se de que pelo menos 2 [nós do trabalhador de borda](cs_edge.html#edge) estejam ativados em cada zona. Se os nós do trabalhador de borda forem ativados em algumas zonas, mas não em outras, os balanceadores de carga não serão implementados uniformemente. Os balanceadores de carga serão implementados em nós de borda em algumas zonas, mas em nós do trabalhador regulares em outras zonas.
  * Se você tem múltiplas VLANs para um cluster, múltiplas sub-redes na mesma VLAN ou um cluster multizona, deve-se ativar o [VLAN Spanning](/docs/infrastructure/vlans/vlan-spanning.html#vlan-spanning) para sua conta de infraestrutura do IBM Cloud (SoftLayer) para que os nós do trabalhador possam se comunicar entre si na rede privada. Para executar essa ação, você precisa da [permissão de infraestrutura](cs_users.html#infra_access) **Rede > Gerenciar rede VLAN Spanning** ou é possível solicitar ao proprietário da conta para ativá-la. Para verificar se o VLAN Spanning já está ativado, use o [comando](/docs/containers/cs_cli_reference.html#cs_vlan_spanning_get) `ibmcloud ks vlan-spanning-get`. Se você está usando o {{site.data.keyword.BluDirectLink}}, deve-se usar um [ Virtual Router Function (VRF)](/docs/infrastructure/direct-link/subnet-configuration.html#more-about-using-vrf). Para ativar o VRF, entre em contato com o representante de conta da infraestrutura do IBM Cloud (SoftLayer).


Para configurar um serviço de balanceador de carga em um cluster de múltiplas zonas:
1.  [Implemente o seu app no cluster](cs_app.html#app_cli). Quando você implementa o seu app no cluster, são criados para você um ou mais pods que executam o seu app em um contêiner. Certifique-se de incluir um rótulo à sua implementação na seção de metadados de seu arquivo de configuração. Esse rótulo é necessário para identificar todos os pods nos quais seu app está sendo executado para que eles possam ser incluídos no balanceamento de carga.

2.  Crie um serviço de balanceador de carga para o app que você deseja expor. Para tornar seu app disponível na Internet pública ou em uma rede privada, crie um serviço do Kubernetes para seu app. Configure seu serviço para incluir todos os pods que compõem o seu app no balanceamento de carga.
  1. Crie um arquivo de configuração de serviço que seja chamado, por exemplo, de `myloadbalancer.yaml`.
  2. Defina um serviço de balanceador de carga para o app que você deseja expor. É possível especificar uma zona e um endereço IP.

      ```
      apiVersion: v1
      kind: Service
      metadata:
        name: myloadbalancer
        annotations:
          service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: <public_or_private>
          service.kubernetes.io/ibm-load-balancer-cloud-provider-zone: "<zone>"
      spec:
        type: LoadBalancer
        selector:
          <selector_key>: <selector_value>
        ports:
         - protocol: TCP
             port: 8080
          loadBalancerIP: <IP_address>
      ```
      {: codeblock}

      <table>
      <caption>Entendendo os componentes de arquivo YAML</caption>
      <thead>
      <th colspan=2><img src="images/idea.png" alt="Ícone de ideia"/> entendendo os componentes de arquivo do YAML</th>
      </thead>
      <tbody>
      <tr>
        <td><code>service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type:</code>
        <td>Anotação para especificar o tipo de balanceador de carga. Os valores aceitos são <code>private</code> e <code>public</code>. Se você estiver criando um balanceador de carga público em clusters em VLANs públicas, essa anotação não será necessária.</td>
      </tr>
      <tr>
        <td><code>service.kubernetes.io/ibm-load-balancer-cloud-provider-zone:</code>
        <td>Anotação para especificar a zona na qual o serviço de balanceador de carga é implementado. Para ver zonas, execute <code>ibmcloud ks zones</code>.</td>
      </tr>
      <tr>
        <td><code>seletor</code></td>
        <td>Insira o par de chave de etiqueta (<em>&lt;selector_key&gt;</em>) e valor (<em>&lt;selector_value&gt;</em>) a serem usados para direcionar os pods nos quais seu app é executado. Para destinar os seus pods e incluí-los no balanceamento de carga de serviço, verifique os valores <em>&lt;selectorkey&gt;</em> e <em>&lt;selectorvalue&gt;</em>. Certifique-se de que sejam iguais ao par de <em>chave/valor</em> usado na seção <code>spec.template.metadata.labels</code> do yaml de sua implementação.</td>
      </tr>
      <tr>
        <td><code>port</code></td>
        <td>A porta na qual o serviço atende.</td>
      </tr>
      <tr>
        <td><code>loadBalancerIP</code></td>
        <td>Opcional: para criar um balanceador de carga privado ou para usar um endereço IP portátil específico para um balanceador de carga público, substitua <em>&lt;IP_address&gt;</em> pelo endereço IP que você deseja usar. Se você especificar uma zona, o endereço IP deverá estar nessa VLAN ou zona. Se você não especificar um endereço IP:<ul><li>Se o seu cluster estiver em uma VLAN pública, um endereço IP móvel público será usado. A maioria dos clusters está em uma VLAN pública.</li><li>Se o seu cluster estiver disponível apenas em uma VLAN privada, então, um endereço IP móvel privado será usado.</li></td>
      </tr>
      </tbody></table>

      Exemplo de arquivo de configuração para criar um serviço de balanceador de carga clássico privado que usa um endereço IP especificado em `dal12`:

      ```
      apiVersion: v1
      kind: Service
      metadata:
        name: myloadbalancer
        annotations:
          service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: private
          service.kubernetes.io/ibm-load-balancer-cloud-provider-zone: "dal12"
      spec:
        type: LoadBalancer
        selector:
          app: nginx
        ports:
         - protocol: TCP port: 8080 loadBalancerIP: 172.21.xxx.xxx
      ```
      {: codeblock}

  3. Opcional: configure um firewall especificando o `loadBalancerSourceRanges` na seção **spec**. Para obter mais informações, veja a [documentação do Kubernetes ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/tasks/access-application-cluster/configure-cloud-provider-firewall/).

  4. Crie o serviço em seu cluster.

      ```
      kubectl apply -f myloadbalancer.yaml
      ```
      {: pre}

3. Verifique se o serviço de balanceador de carga foi criado com êxito. Substitua _&lt;myservice&gt;_ pelo nome do serviço de balanceador de carga que você criou na etapa anterior.

    ```
    Kubectl describe myloadbalancer de serviço
    ```
    {: pre}

    **Nota:** pode levar alguns minutos para o serviço de balanceador de carga ser criado corretamente e para que o app fique disponível.

    Exemplo de saída da CLI:

    ```
    Name:                   myloadbalancer
    Namespace:              default
    Labels:                 <none>
    Selector:               app=liberty
    Type:                   LoadBalancer
    Zone:                   dal10
    IP:                     172.21.xxx.xxx
    LoadBalancer Ingress:   169.xx.xxx.xxx
    Port:                   <unset> 8080/TCP
    NodePort:               <unset> 32040/TCP
    Endpoints:              172.30.xxx.xxx:8080
    Session Affinity:       None
    Events:
      FirstSeen	LastSeen	Count	From			SubObjectPath	Type	 Reason			          Message
      ---------	--------	-----	----			-------------	----	 ------			          -------
      10s		 10s		 1	 {service-controller }	 Normal CreatingLoadBalancer	Creating load balancer 10s		 10s		 1	 {service-controller }		Normal CreatedLoadBalancer	Created load balancer
    ```
    {: screen}

    O endereço IP do **Ingresso de LoadBalancer** é o endereço IP móvel que foi designado ao seu serviço de balanceador de carga.

4.  Se você tiver criado um balanceador de carga público, acesse seu app pela Internet.
    1.  Abra seu navegador da web preferencial.
    2.  Insira o endereço IP público móvel do balanceador de carga e a porta.

        ```
        Http://169.xx.xxx.xxx:8080
        ```
        {: codeblock}        

5. Se você optar por [ativar a preservação do IP de origem para um serviço de balanceador de carga ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-typeloadbalancer), assegure-se de que os pods do app sejam planejados para os nós do trabalhador de borda [incluindo a afinidade do nó de borda nos pods do app](cs_loadbalancer.html#edge_nodes). Os pods do app devem ser planejados nos nós de borda para obter solicitações recebidas.

6. Para manipular solicitações recebidas para seu app de outras zonas, repita as etapas acima para incluir um balanceador de carga em cada zona.

7. Opcional: um serviço do balanceador de carga também disponibiliza o seu app sobre as NodePorts do serviço. [NodePorts](cs_nodeport.html) são acessíveis em cada endereço IP público e privado para cada nó dentro do cluster. Para bloquear o tráfego para NodePorts enquanto você está usando um serviço de balanceador de carga, consulte [Controlando o tráfego de entrada para os serviços de balanceador de carga ou do NodePort](cs_network_policy.html#block_ingress).

## Ativando o acesso público ou privado para um app em um cluster de zona única
{: #config}

Antes de iniciar:

-   Este recurso está disponível somente para clusters padrão.
-   Deve-se ter um endereço IP público ou privado móvel disponível para designar ao serviço de balanceador de carga.
-   Um serviço de balanceador de carga com um endereço IP privado móvel ainda tem um NodePort público aberto em cada nó do trabalhador. Para incluir uma política de rede para evitar o tráfego público, consulte [Bloqueando tráfego recebido](cs_network_policy.html#block_ingress).

Para criar um serviço de balanceador de carga:

1.  [Implemente o seu app no cluster](cs_app.html#app_cli). Quando você implementa o seu app no cluster, são criados para você um ou mais pods que executam o seu app em um contêiner. Certifique-se de incluir um rótulo à sua implementação na seção de metadados de seu arquivo de configuração. Esse rótulo é necessário para identificar todos os pods nos quais o seu app está em execução para que eles possam ser incluídos no balanceamento de carga.
2.  Crie um serviço de balanceador de carga para o app que você deseja expor. Para tornar seu app disponível na Internet pública ou em uma rede privada, crie um serviço do Kubernetes para seu app. Configure seu serviço para incluir todos os pods que compõem o seu app no balanceamento de carga.
    1.  Crie um arquivo de configuração de serviço que seja chamado, por exemplo, de `myloadbalancer.yaml`.

    2.  Defina um serviço de balanceador de carga para o app que você deseja expor.
        ```
        apiVersion: v1
        kind: Service
        metadata:
          name: myloadbalancer
          annotations:
            service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: <public_or_private>
        spec:
          type: LoadBalancer
          selector:
            <selector_key>: <selector_value>
          ports:
           - protocol: TCP
             port: 8080
          loadBalancerIP: <IP_address>
        ```
        {: codeblock}

        <table>
        <caption>Entendendo os componentes de arquivo YAML</caption>
        <thead>
        <th colspan=2><img src="images/idea.png" alt="Ícone de ideia"/> entendendo os componentes de arquivo do YAML</th>
        </thead>
        <tbody>
        <tr>
          <td>`service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type:`
          <td>Anotação para especificar o tipo de balanceador de carga. Os valores aceitos são `private` e `public`. Se você estiver criando um balanceador de carga público em clusters em VLANs públicas, essa anotação não será necessária.</td>
        </tr>
        <tr>
          <td><code>seletor</code></td>
          <td>Insira o par de chave de etiqueta (<em>&lt;selector_key&gt;</em>) e valor (<em>&lt;selector_value&gt;</em>) a serem usados para direcionar os pods nos quais seu app é executado. Para destinar seus pods e incluí-los no balanceamento de carga de serviço, verifique os valores <em>&lt;selector_key&gt;</em> e <em>&lt;selector_value&gt;</em>. Certifique-se de que sejam iguais ao par de <em>chave/valor</em> usado na seção <code>spec.template.metadata.labels</code> do yaml de sua implementação.</td>
        </tr>
        <tr>
          <td><code>port</code></td>
          <td>A porta na qual o serviço atende.</td>
        </tr>
        <tr>
          <td><code>loadBalancerIP</code></td>
          <td>Opcional: para criar um balanceador de carga privado ou para usar um endereço IP portátil específico para um balanceador de carga público, substitua <em>&lt;IP_address&gt;</em> pelo endereço IP que você deseja usar. Se você não especificar um endereço IP:<ul><li>Se o seu cluster estiver em uma VLAN pública, um endereço IP móvel público será usado. A maioria dos clusters está em uma VLAN pública.</li><li>Se o seu cluster estiver disponível apenas em uma VLAN privada, então, um endereço IP móvel privado será usado.</li></td>
        </tr>
        </tbody></table>

        Exemplo de arquivo de configuração para criar um serviço de balanceador de carga clássico privado que usa um endereço IP especificado:

        ```
        apiVersion: v1
        kind: Service
        metadata:
          name: myloadbalancer
          annotations:
            service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: private
        spec:
          type: LoadBalancer
          selector:
            app: nginx
          ports:
           - protocol: TCP port: 8080 loadBalancerIP: 172.21.xxx.xxx
        ```
        {: codeblock}

    3.  Opcional: configure um firewall especificando o `loadBalancerSourceRanges` na seção **spec**. Para obter mais informações, veja a [documentação do Kubernetes ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/tasks/access-application-cluster/configure-cloud-provider-firewall/).

    4.  Crie o serviço em seu cluster.

        ```
        kubectl apply -f myloadbalancer.yaml
        ```
        {: pre}

        Quando o serviço de balanceador de carga for criado, um endereço IP móvel será designado automaticamente ao balanceador de carga. Se nenhum endereço IP móvel estiver disponível, o serviço de balanceador de carga não poderá ser criado.

3.  Verifique se o serviço de balanceador de carga foi criado com êxito.

    ```
    Kubectl describe myloadbalancer de serviço
    ```
    {: pre}

    **Nota:** pode levar alguns minutos para o serviço de balanceador de carga ser criado corretamente e para que o app fique disponível.

    Exemplo de saída da CLI:

    ```
    Name: myloadbalancer Namespace: default Labels: <none> Selector: app=liberty Type: LoadBalancer Location: dal10 IP: 172.21.xxx.xxx LoadBalancer Ingress: 169.xx.xxx.xxx Port: <unset> 8080/TCP NodePort: <unset> 32040/TCP Endpoints: 172.30.xxx.xxx:8080 Session Affinity: None Events: FirstSeen	LastSeen	Count	From			SubObjectPath	Type	 Reason			 Message
      ---------	--------	-----	----			-------------	----	 ------			          -------
      10s		 10s		 1	 {service-controller }	 Normal CreatingLoadBalancer	Creating load balancer 10s		 10s		 1	 {service-controller }		Normal CreatedLoadBalancer	Created load balancer
    ```
    {: screen}

    O endereço IP do **Ingresso de LoadBalancer** é o endereço IP móvel que foi designado ao seu serviço de balanceador de carga.

4.  Se você tiver criado um balanceador de carga público, acesse seu app pela Internet.
    1.  Abra seu navegador da web preferencial.
    2.  Insira o endereço IP público móvel do balanceador de carga e a porta.

        ```
        Http://169.xx.xxx.xxx:8080
        ```
        {: codeblock}

5. Se você optar por [ativar a preservação do IP de origem para um serviço de balanceador de carga ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-typeloadbalancer), assegure-se de que os pods do app sejam planejados para os nós do trabalhador de borda [incluindo a afinidade do nó de borda nos pods do app](cs_loadbalancer.html#edge_nodes). Os pods do app devem ser planejados nos nós de borda para obter solicitações recebidas.

6. Opcional: um serviço do balanceador de carga também disponibiliza o seu app sobre as NodePorts do serviço. [NodePorts](cs_nodeport.html) são acessíveis em cada endereço IP público e privado para cada nó dentro do cluster. Para bloquear o tráfego para NodePorts enquanto você está usando um serviço de balanceador de carga, consulte [Controlando o tráfego de entrada para os serviços de balanceador de carga ou do NodePort](cs_network_policy.html#block_ingress).

<br />


## Incluindo a afinidade de nó e as tolerâncias nos pods do app para o IP de origem
{: #node_affinity_tolerations}

Quando uma solicitação do cliente para seu app é enviada para seu cluster, a solicitação é roteada para o pod de serviço de balanceador de carga que expõe seu app. Se nenhum pod de app existir no mesmo nó do trabalhador que o pod de serviço de balanceador de carga, o balanceador de carga encaminhará a solicitação para um pod de app em um nó do trabalhador diferente. O endereço IP de origem do pacote é mudado para o endereço IP público do nó do trabalhador no qual o pod de app está em execução.

Para preservar o endereço IP de origem original da solicitação do cliente, é possível [ativar o IP de origem ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip) para os serviços de balanceador de carga. A conexão TCP continua todo o caminho para os pods de app para que o app possa ver o endereço IP de origem real do inicializador. Preservar o IP do cliente é útil, por exemplo, quando os servidores de app precisam aplicar as políticas de segurança e de controle de acesso.

Após a ativação do IP de origem, os pods do serviço de balanceador de carga devem encaminhar solicitações para pods do app que são implementados no mesmo nó do trabalhador somente. Geralmente, os pods de serviço de balanceador de carga também são implementados para os nós do trabalhador nos quais os pods de app são implementados. No entanto, existem algumas situações em que os pods do balanceador de carga e os pods do app podem não ser planejados no mesmo nó do trabalhador:

* Você tem nós de borda que estão contaminados para que somente pods do serviço de balanceador de carga possam ser implementados neles. Pods do app não podem ser implementados nesses nós.
* O seu cluster está conectado a múltiplas VLANs públicas ou privadas e os pods de seu app podem ser implementados em nós do trabalhador conectados apenas a uma VLAN. Os pods do serviço de balanceador de carga podem não ser implementados nesses nós do trabalhador porque o endereço IP do balanceador de carga está conectado a uma VLAN diferente daquela dos nós do trabalhador.

Para forçar seu app a ser implementado em nós do trabalhador específicos nos quais os pods de serviço de balanceador de carga também podem ser implementados, deve-se incluir regras de afinidade e tolerâncias em sua implementação do app.

### Incluindo regras de afinidade e tolerâncias do nó de borda
{: #edge_nodes}

Quando você [rotula os nós do trabalhador como nós de borda](cs_edge.html#edge_nodes) e também [contamina os nós de borda](cs_edge.html#edge_workloads), os pods do serviço de balanceador de carga são implementados somente nesses nós de borda e os pods de app não podem ser implementados nos nós de borda. Quando o IP de origem for ativado para o serviço de balanceador de carga, os pods do balanceador de carga nos nós de borda não poderão encaminhar solicitações recebidas para seus pods de app em outros nós do trabalhador.
{:shortdesc}

Para forçar a implementação dos pods de app nos nós de borda, inclua uma [regra de afinidade ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature) e uma [tolerância ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/#concepts) do nó de borda na implementação do app.

Exemplo de yaml de implementação com afinidade do nó de borda e tolerância do nó de borda:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: with-node-affinity
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: dedicated
                operator: In
                values:
                - edge
      tolerations:
        - key: dedicated
          value: edge
...
```
{: codeblock}

As seções **affinity** e **tolerations** têm `dedicated` como o `key` e `edge` como o `value`.

### Incluindo regras de afinidade para múltiplas VLANs públicas ou privadas
{: #edge_nodes_multiple_vlans}

Quando o cluster está conectado a múltiplas VLANs públicas ou privadas, os pods do app podem ser implementados nos nós do trabalhador que são conectados apenas a uma VLAN. Se o endereço IP do balanceador de carga estiver conectado a uma VLAN diferente desses nós do trabalhador, os pods do serviço de balanceador de carga não serão implementados nesses nós do trabalhador.
{:shortdesc}

Quando o IP de origem for ativado, planeje os pods de app nos nós do trabalhador que forem a mesma VLAN que o endereço IP do balanceador de carga, incluindo uma regra de afinidade na implementação do app.

Antes de iniciar: [Efetue login em sua conta. Destine a região apropriada e, se aplicável, o grupo de recursos. Configure o contexto para seu cluster](cs_cli_install.html#cs_cli_configure).

1. Obtenha o endereço IP do serviço de balanceador de carga. Procure o endereço IP no campo **Ingress do LoadBalancer**.
    ```
    kubectl describe service <loadbalancer_service_name>
    ```
    {: pre}

2. Recupere o ID da VLAN a que o serviço de balanceador de carga está conectado.

    1. Liste as VLANs públicas móveis para seu cluster.
        ```
        ibmcloud ks cluster-get < cluster_name_or_ID> -- showResources
        ```
        {: pre}

        Saída de exemplo:
        ```
        ...

        Subnet VLANs
        VLAN ID   Subnet CIDR       Public   User-managed
        2234947   10.xxx.xx.xxx/29  false    false
        2234945   169.36.5.xxx/29   true     false
        ```
        {: screen}

    2. Na saída sob **VLANs da sub-rede**, procure o CIDR de sub-rede que corresponde ao endereço IP do balanceador de carga recuperado anteriormente e anote o ID da VLAN.

        Por exemplo, se o endereço IP do serviço de balanceador de carga for `169.36.5.xxx`, a sub-rede correspondente na saída de exemplo da etapa anterior será `169.36.5.xxx/ 29`. O ID da VLAN ao qual a sub-rede está conectada é `2234945`.

3. [Inclua uma regra de afinidade ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#node-affinity-beta-feature) na implementação do app para o ID da VLAN que você anotou na etapa anterior.

    Por exemplo, se você tiver múltiplas VLANs, mas quiser que os pods de seu app sejam implementados nos nós do trabalhador somente na VLAN pública `2234945`:

    ```
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: with-node-affinity
    spec:
      template:
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: publicVLAN
                    operator: In
                    values:
                    - "2234945"
    ...
    ```
    {: codeblock}

    No YAML de exemplo, a seção **affinity** tem `publicVLAN` como o `chave` e `"2234945"` como o `value`.

4. Aplique o arquivo de configuração de implementação atualizado.
    ```
    Kubectl apply -f com-node-affinity.yaml
    ```
    {: pre}

5. Verifique se os pods do app implementados nos nós do trabalhador estão conectados à VLAN designada.

    1. Liste os pods em seu cluster. Substitua `<selector>` pelo rótulo que você usou para o app.
        ```
        kubectl get pods -o wide app=<selector>
        ```
        {: pre}

        Saída de exemplo:
        ```
        NAME                   READY     STATUS              RESTARTS   AGE       IP               NODE
        cf-py-d7b7d94db-vp8pq  1/1       Running             0          15d       172.30.xxx.xxx   10.176.48.78
        ```
        {: screen}

    2. Na saída, identifique um pod para seu app. Observe o ID de **NÓ** do nó do trabalhador em que o pod está.

        Na saída de exemplo da etapa anterior, o pod de app `cf-py-d7b7d94db-vp8pq` está no nó do trabalhador `10.176.48.78`.

    3. Liste os detalhes para o nó do trabalhador.

        ```
        kubectl describe node <worker_node_ID>
        ```
        {: pre}

        Saída de exemplo:

        ```
        Name:                   10.xxx.xx.xxx
        Role:
        Labels:                 arch=amd64
                                beta.kubernetes.io/arch=amd64
                                beta.kubernetes.io/os=linux
                                failure-domain.beta.kubernetes.io/region=us-south
                                failure-domain.beta.kubernetes.io/zone=dal10
                                ibm-cloud.kubernetes.io/encrypted-docker-data=true
                                kubernetes.io/hostname=10.xxx.xx.xxx
                                privateVLAN=2234945
                                publicVLAN=2234967
        ...
        ```
        {: screen}

    4. Na seção **Rótulos** da saída, verifique se a VLAN pública ou privada é a VLAN que você designou nas etapas anteriores.

