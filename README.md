# Experimenting-Openflow-with-mininet
from pox.core import core
import pox.openflow.libopenflow_01 as of

log = core.getLogger()

class Tutorial (object):
  def __init__ (self, connection):
    # Keep track of the connection to the switch so that we can
    # send it messages!
    self.connection = connection

    # This binds our PacketIn event listener
    connection.addListeners(self)
        # Use this table to keep track of which ethernet address is on
    # which switch port (keys are MACs, values are ports).
    self.mac_to_port = {}

  def resend_packet (self, packet_in, out_port):
    """
    Instructs the switch to resend a packet that it had sent to us.
    "packet_in" is the ofp_packet_in object the switch had sent to the
    controller due to a table-miss.
    """
    msg = of.ofp_packet_out()
    msg.data = packet_in

    # Add an action to send to the specified port
    action = of.ofp_action_output(port = out_port)
    msg.actions.append(action)

    # Send message to switch
    self.connection.send(msg)
    
    def act_like_switch (self, packet, packet_in):
    if packet.src not in self.mac_to_port:
        print "Learning that " + str(packet.src) + " is attached at port " + str(packet_in.in_port)
        self.mac_to_port[packet.src] = packet_in.in_port
        msg=of.ofp_flow_mod()
        msg.match.dl_dst=packet.src
        out_action=of.ofp_action_output(port=packet_in.in_port)
        msg.actions.append(out_action)
        self.connection.send(msg)

    if packet.dst in self.mac_to_port:
      print str(packet.dst) + " destination known. only send message to it"
      self.resend_packet(packet_in, self.mac_to_port[packet.dst])

    else:
      print str(packet.dst) + " not known, resend to everybody"
      self.resend_packet(packet_in, of.OFPP_ALL)

  def _handle_PacketIn (self, event):
    packet = event.parsed # This is the parsed packet data.
    if not packet.parsed:
      log.warning("Ignoring incomplete packet")
      return

    packet_in = event.ofp # The actual ofp_packet_in message.
    #self.act_like_hub(packet, packet_in)
    self.act_like_switch(packet, packet_in)

def launch ():
  
  def start_switch (event):
    log.debug("Controlling %s" % (event.connection,))
    Tutorial(event.connection)
  core.openflow.addListenerByName("ConnectionUp", start_switch)
                                                                                         
