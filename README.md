# FARA-TIME
Clock-In
<script>
  async function send(type){
    await fetch("https://gbwirxwapmkvziucoysh.supabase.co/rest/v1/time_records", {
      method: "POST",
      headers: {
        "apikey": "sb_publishable_aVx3qUNe8IJ7ZwzetMWxKw_qgxE0VNS",
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        employee_id: "test",
        employee_name: "Test User",
        type: type,
        timestamp: new Date().toISOString()
      })
    });
    alert(type + " sent!");
  }

  function clockIn(){ send("clock_in"); }
  function clockOut(){ send("clock_out"); }
</script>
