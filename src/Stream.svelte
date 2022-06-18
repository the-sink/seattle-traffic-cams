<script>
    import { Card, CardBody } from 'sveltestrap';

    export let stream;
</script>

<Card class="w-25">
    <CardBody>
        <video autoplay controls id={stream} class="streamPlayer" style="width:100%;height:100%;object-fit:cover;"><track kind="captions"></video>
    </CardBody>
</Card>